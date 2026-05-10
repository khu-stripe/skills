# Auth Rate Analysis — Field Reference

## Primary Table: `fact.payments_view`

Trino view of the canonical payments schema. Fresher than `fact.payments` by ~10 hours.

**Always apply these base filters:**
```sql
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND charge_has_gateway_conversation = true  -- Only charges that reached the issuer
  AND is_card_validation = false               -- Exclude $0 card validations
```

---

## Auth Rate Calculation

```
Auth Rate = authorized / (authorized + issuer_declined)
```

Use `charge_outcome__type`:
| Value | Meaning | Include in denominator? |
|-------|---------|------------------------|
| `authorized` | Issuer approved | ✅ Yes (numerator + denominator) |
| `issuer_declined` | Issuer rejected | ✅ Yes (denominator only) |
| `blocked` | Blocked by Stripe (Radar/Shield) | ❌ No — not an issuer decision |
| `invalid` | Invalid request | ❌ No — not an issuer decision |

**Important:** `charge_has_gateway_conversation = true` already excludes most blocked/invalid charges, but some edge cases exist. The FILTER approach (counting only `authorized` and `issuer_declined`) is safest.

---

## Key Fields

### Outcome Fields

| Field | Type | Description |
|-------|------|-------------|
| `charge_outcome__type` | varchar | `authorized`, `issuer_declined`, `blocked`, `invalid` |
| `charge_outcome__reason` | varchar | User-facing decline reason: `do_not_honor`, `insufficient_funds`, `incorrect_cvc`, etc. |
| `charge_outcome__derived` | boolean | If true, outcome was derived internally (not from mongo.chargeoutcomes) |
| `gateway_conversation__response_code` | varchar | Raw acquirer/issuer response code (e.g., `59`, `51`, `N7`) |

### 3DS Fields

| Field | Type | Description |
|-------|------|-------------|
| `three_d_secure_outcome__version` | varchar | 3DS protocol version (`2.2.0`, `1.0.2`, etc.). NULL = no 3DS |
| `three_d_secure_outcome__result` | varchar | `authenticated`, `attempt_acknowledged`, `failed`, `rejected` |
| `three_d_secure_outcome__authentication_flow` | varchar | `challenge`, `frictionless`. **NULL if externally imported** |
| `three_d_secure_outcome__api` | varchar | API call that triggered 3DS |

**Detecting external 3DS import:**
- `three_d_secure_outcome__version IS NOT NULL` (3DS was used)
- `three_d_secure_outcome__authentication_flow IS NULL` (no Stripe authentication flow)
- This means the merchant imported 3DS results via `payment_method_options.card.three_d_secure`

**Fields NOT available on `fact.payments_view`** (learned from column errors):
- ❌ `three_d_secure_outcome__electronic_commerce_indicator` (ECI)
- ❌ `three_d_secure_outcome__result_reason`
- ❌ `liability_shift_is_successful`

### Geography Fields

| Field | Type | Description |
|-------|------|-------------|
| `merchant_country` | varchar | Country of the merchant (settlement entity) |
| `card_bin_country` | varchar | Country from the card BIN table |
| `gateway_country` | varchar | Country of the settlement merchant / card acceptor |
| `charge_country` | varchar | Derived charge country (complex rules for Connect) |

**Domestic vs Cross-border:**
```sql
CASE WHEN card_bin_country = merchant_country THEN 'Domestic' ELSE 'Cross-border' END
```

### Card Fields

| Field | Type | Description |
|-------|------|-------------|
| `card_brand` | varchar | `visa`, `mc`, `amex`, `dscvr`, `jcb`, `diners` |
| `card_bin` | varchar | Full BIN (use `SUBSTR(card_bin, 1, 6)` for 6-digit BIN) |
| `card_bin_description` | varchar | Issuer description from BIN table |
| `card_bin_bank` | varchar | Issuing bank name |
| `payment_instrument` | varchar | `credit`, `debit`, `prepaid`, `ach`, `unknown` |
| `payment_method` | varchar | `card_charge`, `card_present`, `apple_pay`, `google`, etc. |

### AVS Fields

| Field | Type | Description |
|-------|------|-------------|
| `gateway_conversation__response_interpreted_avs_zip_postal` | varchar | **Zip/postal check result** (`pass`, `fail`, `unchecked`). **This is what Auth Dash shows.** |
| `gateway_conversation__response_interpreted_avs_address` | varchar | Street address check result (`pass`, `fail`, `unchecked`). NOT what Auth Dash shows by default. |
| `gateway_conversation__response_avs_response` | varchar | Raw AVS response code (Y, Z, N, A, U, S, R, W, X, etc.) |
| `gateway_conversation__request_avs_address` | varchar | The address string sent to the acquirer |

**⚠️ Common mistake:** Using `avs_address` (street check) instead of `avs_zip_postal` (zip check). Auth Dash uses zip. Most US issuers weight zip much more than street.

### Integration Fields

| Field | Type | Description |
|-------|------|-------------|
| `is_stripejs` | boolean | Payment created using Stripe.js (includes dashboard, checkout) |
| `stripe_ui` | varchar | Surface used to collect payment method |
| `stripe_js_version` | varchar | Major Stripe.js version (V2 legacy, V3 Elements) |
| `gateway_conversation__request_cvc` | boolean | Whether CVC was sent to acquirer |
| `shield_action` | varchar | Radar action taken (populated = Radar active) |

### Amount Fields

| Field | Type | Description |
|-------|------|-------------|
| `auth_presentment__amount_in_usd` | bigint | Auth amount in USD **cents** (divide by 100) |
| `captured__amount_in_usd` | bigint | Captured amount in USD **cents** |

### Time Fields

| Field | Type | Description |
|-------|------|-------------|
| `created` | timestamp | When the payment object was created |
| `capture_date` | timestamp | When captured (only for successful payments) |

---

## Domain Knowledge: 3DS Import Issues

### ECI Values (Critical for External 3DS Import)

| Network | Authenticated (full) | Attempted (not enrolled) |
|---------|----------------------|--------------------------|
| **Visa** | `05` | `06` |
| **Mastercard** | `02` | `01` |
| **Amex** | `05` | `06` |

**Visa and MC use DIFFERENT ECI values.** Mixing them up is the #1 cause of imported 3DS hurting auth rates.

### Common 3DS Import Mistakes

1. **Wrong ECI value** — e.g., sending Visa ECI `02` (MC convention) instead of `05`
2. **Stale cryptogram** — cryptograms expire after 15-30 minutes at most issuers
3. **Truncated/padded cryptogram** — must be exactly 28 chars (base64 of 20 bytes)
4. **Missing `ares_trans_status`** — should be `Y` for authenticated, `A` for attempt
5. **`transaction_id` mismatch** — dsTransID must match between AReq/ARes and authorization

### Diagnostic Signals for 3DS Import Quality

| Signal | Interpretation |
|--------|---------------|
| `attempt_acknowledged` outperforms `authenticated` | Cryptogram/ECI for authenticated charges is being rejected |
| Post-auth decline rate >20% for `authenticated` | Highly abnormal — issuer authenticates but still declines |
| `do_not_honor` (code 59) dominant after 3DS auth | Issuer doesn't trust the 3DS data |
| `processing_error` with no raw code after 3DS auth | Acquirer/issuer couldn't parse the 3DS payload |
| Same BIN: 3DS auth rate < no-3DS auth rate by >15pp | Conclusive evidence of 3DS data quality issue |
| 3DS hurts domestic but helps cross-border | US issuers reject bad cryptograms; XB issuers reward any 3DS |

### 3DS Import API Fields

Required fields in `payment_method_options.card.three_d_secure`:
| Field | Required | What to Check |
|-------|----------|---------------|
| `cryptogram` | ✅ | Exact CAVV/AAV from external MPI. 20 bytes, base64 (28 chars). Must be fresh. |
| `transaction_id` | ✅ | Exact `dsTransID` from 3DS2 authentication |
| `version` | ✅ | Must match actual protocol version (e.g., `2.2.0`) |
| `electronic_commerce_indicator` | ✅ | Correct per card brand — see ECI table above |
| `ares_trans_status` | Recommended | `Y` = authenticated, `A` = attempted |
| `error_on_requires_action` | Recommended | Set `true` to prevent Stripe re-triggering 3DS |

---

## Decline Reason Glossary

| Reason | Raw Codes | Actionable? | Notes |
|--------|-----------|-------------|-------|
| `do_not_honor` | 05, 59 | Limited | Generic issuer refusal. Adaptive Acceptance helps. |
| `insufficient_funds` | 51 | No | Cardholder issue. More common on debit/prepaid. |
| `incorrect_cvc` | N7 | Yes | CVC didn't match. Client-side validation helps. |
| `invalid_cvc` | 82 | Yes | CVC format invalid. Client-side validation helps. |
| `generic_decline` | 70, 83 | Limited | Catch-all. Adaptive Acceptance helps. |
| `transaction_not_allowed` | 5C, 62, 78, 9G | Investigate | Card restrictions, MCC issues, or geographic blocks. |
| `processing_error` | various | Investigate | Check for 3DS payload issues if after 3DS auth. |
| `try_again_later` | 79, 82, 83 | Retry | Temporary issuer issue. Smart retries help. |
| `card_velocity_exceeded` | 61 | No | Too many transactions in short time. |
| `expired_card` | 54 | No | Card is expired. Prompt for new card. |

---

## Internal Resources

| Resource | Link |
|----------|------|
| Auth Dash 360 | [go/authdash360](https://go/authdash360) |
| AuthOpt Workbook | [go/one-click](https://go/one-click) |
| Authorization FAQs | [Trailhead](https://trailhead.corp.stripe.com/docs/global-payments-performance-team/authorization-faqs) |
| Network Decline Codes | [go/networkdeclinecodes](https://go/networkdeclinecodes) |
| 3DS Import Docs | [docs.stripe.com](https://docs.stripe.com/payments/payment-intents/three-d-secure-import) |
| Payments Performance (questions) | #payments-performance-ask |
| fact.payments_view catalog | [Hubble](https://hubble.corp.stripe.com/viz/catalog/datasets/fact.payments_view) |
