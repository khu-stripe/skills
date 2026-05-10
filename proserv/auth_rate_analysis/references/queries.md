# Auth Rate Analysis — SQL Query Templates

All queries target `fact.payments_view` (Trino). Replace `{{MERCHANT_ID}}`, `{{SINCE_START}}`, `{{SINCE_END}}`, `{{BEFORE_START}}` with actual values.

**Base filters (apply to ALL queries):**
```sql
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND charge_has_gateway_conversation = true
  AND is_card_validation = false
```

**Auth rate formula:**
```sql
ROUND(
  CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE)
  / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2
) AS auth_rate
```

---

## Query 0: Integration Profile

Identifies the merchant's integration shape. Run first to determine which deep-dives are relevant.

```sql
SELECT
  is_stripejs,
  stripe_ui,
  stripe_js_version,
  three_d_secure_outcome__version AS tds_version,
  three_d_secure_outcome__authentication_flow AS tds_auth_flow,
  three_d_secure_outcome__result AS tds_result,
  payment_method,
  payment_instrument,
  gateway_conversation__request_cvc AS cvc_sent,
  shield_action,
  CASE WHEN card_bin_country = merchant_country THEN 'Domestic' ELSE 'Cross-border' END AS geo,
  COUNT(*) AS charges
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00'
  AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true
  AND is_card_validation = false
GROUP BY 1,2,3,4,5,6,7,8,9,10,11
ORDER BY 12 DESC
LIMIT 50
```

**Interpretation:**
- `tds_auth_flow` = NULL + `tds_version` IS NOT NULL → **externally imported 3DS**
- `tds_auth_flow` = 'frictionless' or 'challenge' → Stripe-native 3DS
- `is_stripejs` = false, `stripe_ui` = NULL → server-side API integration
- `shield_action` populated → Radar is active
- `payment_method` = 'card_charge' + `payment_instrument` = 'credit'/'debit' → FPAN card charges

---

## Query 1: Before vs Since — Multi-Dimensional Comparison

Single UNION ALL query producing before/since comparison across 7 dimensions.

```sql
-- OVERALL
SELECT 'OVERALL' AS dimension, 'Total' AS segment,
  CASE WHEN created < TIMESTAMP '{{SINCE_START}} 00:00:00' THEN 'Before' ELSE 'Since' END AS period,
  COUNT(*) AS total_charges,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS authorized,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined') AS declined,
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2) AS auth_rate
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{BEFORE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 3

UNION ALL

-- 3DS STATUS
SELECT '3DS_STATUS',
  CASE WHEN three_d_secure_outcome__version IS NOT NULL THEN 'With 3DS' ELSE 'Without 3DS' END,
  CASE WHEN created < TIMESTAMP '{{SINCE_START}} 00:00:00' THEN 'Before' ELSE 'Since' END,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{BEFORE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2, 3

UNION ALL

-- 3DS RESULT TYPE (imported 3DS detail)
SELECT '3DS_RESULT',
  COALESCE(three_d_secure_outcome__result, 'no_3ds'),
  CASE WHEN created < TIMESTAMP '{{SINCE_START}} 00:00:00' THEN 'Before' ELSE 'Since' END,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{BEFORE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2, 3

UNION ALL

-- GEOGRAPHY (domestic vs cross-border)
SELECT 'GEO',
  CASE WHEN card_bin_country = merchant_country THEN 'Domestic' ELSE 'Cross-border' END,
  CASE WHEN created < TIMESTAMP '{{SINCE_START}} 00:00:00' THEN 'Before' ELSE 'Since' END,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{BEFORE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2, 3

UNION ALL

-- CARD BRAND
SELECT 'BRAND', card_brand,
  CASE WHEN created < TIMESTAMP '{{SINCE_START}} 00:00:00' THEN 'Before' ELSE 'Since' END,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{BEFORE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2, 3

UNION ALL

-- INSTRUMENT (credit / debit / prepaid)
SELECT 'INSTRUMENT', payment_instrument,
  CASE WHEN created < TIMESTAMP '{{SINCE_START}} 00:00:00' THEN 'Before' ELSE 'Since' END,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{BEFORE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2, 3

UNION ALL

-- AVS ZIP (what Auth Dash shows)
SELECT 'AVS_ZIP',
  COALESCE(gateway_conversation__response_interpreted_avs_zip_postal, 'not_sent'),
  CASE WHEN created < TIMESTAMP '{{SINCE_START}} 00:00:00' THEN 'Before' ELSE 'Since' END,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{BEFORE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2, 3

ORDER BY 1, 2, 3
```

---

## Query 2: 3DS × Geography × Card Brand Cross-Tab (Since period only)

```sql
SELECT
  three_d_secure_outcome__result AS tds_result,
  CASE WHEN card_bin_country = merchant_country THEN 'Domestic' ELSE 'Cross-border' END AS geo,
  card_brand,
  COUNT(*) AS charges,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS auth,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined') AS declined,
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2) AS auth_rate
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
  AND three_d_secure_outcome__version IS NOT NULL
GROUP BY 1, 2, 3
ORDER BY 1, 2, 4 DESC
```

---

## Query 3: Post-3DS-Authentication Decline Reasons

Why do charges that passed 3DS still get declined? High post-auth decline rate = cryptogram/ECI quality issue.

```sql
SELECT
  three_d_secure_outcome__result AS tds_result,
  charge_outcome__reason,
  gateway_conversation__response_code AS raw_code,
  COUNT(*) AS cnt
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
  AND three_d_secure_outcome__version IS NOT NULL
  AND charge_outcome__type = 'issuer_declined'
GROUP BY 1, 2, 3
ORDER BY 1, 4 DESC
```

**Key signals:**
- `do_not_honor` (code 59) dominant after `authenticated` → issuer rejects cryptogram/ECI
- `processing_error` with no raw code → acquirer/issuer couldn't parse 3DS data payload
- Post-auth decline rate >20% for `authenticated` is highly abnormal

---

## Query 4: Decline Reasons with Raw Codes (Since period)

```sql
SELECT
  charge_outcome__reason,
  gateway_conversation__response_code AS raw_code,
  COUNT(*) AS cnt,
  ROUND(CAST(COUNT(*) AS DOUBLE) / (
    SELECT COUNT(*) FROM fact.payments_view
    WHERE merchant_id = '{{MERCHANT_ID}}'
      AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
      AND charge_has_gateway_conversation = true AND is_card_validation = false
      AND charge_outcome__type = 'issuer_declined'
  ) * 100, 2) AS pct_of_declines
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
  AND charge_outcome__type = 'issuer_declined'
GROUP BY 1, 2
ORDER BY 3 DESC
```

---

## Query 5: BIN-Level 3DS vs No-3DS Comparison (Smoking Gun)

Compares the **same card BINs** with and without 3DS. A large gap on the same BIN is conclusive evidence of 3DS data quality issues.

```sql
-- With 3DS
SELECT
  '3DS' AS tds_flag,
  SUBSTR(card_bin, 1, 6) AS bin_6,
  card_bin_description AS issuer,
  COUNT(*) AS charges,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS auth,
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2) AS auth_rate
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
  AND three_d_secure_outcome__version IS NOT NULL
GROUP BY 1, 2, 3
HAVING COUNT(*) >= 5

UNION ALL

-- Without 3DS (same BINs)
SELECT
  'No 3DS',
  SUBSTR(card_bin, 1, 6),
  card_bin_description,
  COUNT(*),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
  AND three_d_secure_outcome__version IS NULL
GROUP BY 1, 2, 3
HAVING COUNT(*) >= 5

ORDER BY 2, 1
```

**Interpretation:** If the same BIN has >15pp gap between 3DS and no-3DS auth rate, the imported 3DS data is actively harming auth.

---

## Query 6: AVS Raw Code Breakdown (Full Picture)

Shows both AVS components (street + zip) and the raw AVS code for the complete picture.

```sql
SELECT
  COALESCE(gateway_conversation__response_avs_response, 'null') AS raw_avs_code,
  COALESCE(gateway_conversation__response_interpreted_avs_address, 'null') AS avs_address,
  COALESCE(gateway_conversation__response_interpreted_avs_zip_postal, 'null') AS avs_zip,
  COUNT(*) AS charges,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS auth,
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2) AS auth_rate
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 1, 2, 3
ORDER BY 4 DESC
```

**Common raw AVS codes:**
| Code | Street | Zip | Meaning |
|------|--------|-----|---------|
| Y | ✅ pass | ✅ pass | Full match |
| Z | ❌ fail | ✅ pass | Zip match only (normal for US e-commerce) |
| A | ✅ pass | ❌ fail | Address match, zip mismatch |
| N | ❌ fail | ❌ fail | Neither matches |
| U | unchecked | unchecked | Unavailable (often cross-border) |
| S | unchecked | unchecked | Not supported |
| R | unchecked | unchecked | System unavailable |
| W | unchecked | ✅ pass | Zip match, address N/A |

**Important:** Auth Dash uses `gateway_conversation__response_interpreted_avs_zip_postal` (zip check), NOT the address check. Use `avs_zip` for the metric that matches Auth Dash.

---

## Query 7: 3DS by Transaction Amount (Since period)

```sql
SELECT
  CASE WHEN three_d_secure_outcome__version IS NOT NULL THEN 'With 3DS' ELSE 'No 3DS' END AS tds,
  CASE
    WHEN auth_presentment__amount_in_usd / 100.0 < 50 THEN 'a) <$50'
    WHEN auth_presentment__amount_in_usd / 100.0 < 100 THEN 'b) $50-99'
    WHEN auth_presentment__amount_in_usd / 100.0 < 250 THEN 'c) $100-249'
    WHEN auth_presentment__amount_in_usd / 100.0 < 500 THEN 'd) $250-499'
    WHEN auth_presentment__amount_in_usd / 100.0 < 1000 THEN 'e) $500-999'
    ELSE 'f) $1000+'
  END AS amount_bucket,
  COUNT(*) AS charges,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS auth,
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2) AS auth_rate
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 1, 2
ORDER BY 1, 2
```

---

## Query 8: Daily Auth Rate Trend (Full Window)

```sql
SELECT
  CAST(created AS DATE) AS day,
  COUNT(*) AS total_charges,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS authorized,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined') AS declined,
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2) AS auth_rate
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{BEFORE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 1
ORDER BY 1
```

---

## Query 9: Current-Period Multi-Dimensional Snapshot (Since period)

Single UNION ALL query for the "since" period producing the Section 1 breakdown table.

```sql
SELECT 'OVERALL' AS dimension, 'Total' AS segment,
  COUNT(*) AS total_charges,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS authorized,
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined') AS declined,
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2) AS auth_rate
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false

UNION ALL

SELECT '3DS_STATUS',
  CASE WHEN three_d_secure_outcome__version IS NOT NULL THEN 'With 3DS (imported)' ELSE 'Without 3DS' END,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2

UNION ALL

SELECT 'GEO',
  CASE WHEN card_bin_country = merchant_country THEN 'Domestic' ELSE 'Cross-border' END,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2

UNION ALL

SELECT 'BRAND', card_brand,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2

UNION ALL

SELECT 'INSTRUMENT', payment_instrument,
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2

UNION ALL

SELECT 'AVS_ZIP',
  COALESCE(gateway_conversation__response_interpreted_avs_zip_postal, 'not_sent'),
  COUNT(*), COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized'),
  COUNT(*) FILTER (WHERE charge_outcome__type = 'issuer_declined'),
  ROUND(CAST(COUNT(*) FILTER (WHERE charge_outcome__type = 'authorized') AS DOUBLE) / NULLIF(CAST(COUNT(*) AS DOUBLE), 0) * 100, 2)
FROM fact.payments_view
WHERE merchant_id = '{{MERCHANT_ID}}'
  AND created >= TIMESTAMP '{{SINCE_START}} 00:00:00' AND created < TIMESTAMP '{{SINCE_END}} 00:00:00'
  AND charge_has_gateway_conversation = true AND is_card_validation = false
GROUP BY 2

ORDER BY 1, 3 DESC
```
