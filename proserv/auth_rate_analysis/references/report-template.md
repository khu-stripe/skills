# Auth Rate Report — Markdown Template

Use this structure when generating the report. Replace placeholders with actual data. Omit sections that don't apply (e.g., skip 3DS deep-dive if merchant doesn't use 3DS).

---

```markdown
# Auth Rate Optimization Report — Since {{SINCE_START}}
## Merchant: `{{MERCHANT_ID}}` ({{MERCHANT_COUNTRY}}-based)
## Integration: {{INTEGRATION_SHAPE}}
### Period: {{SINCE_START}}–{{SINCE_END}} | Report Date: {{REPORT_DATE}}

---

# 0. BEFORE vs SINCE — QUICK COMPARISON

> Before = {{BEFORE_START}} – {{BEFORE_END}} ({{BEFORE_DAYS}} days) | Since = {{SINCE_START}} – {{SINCE_END}} ({{SINCE_DAYS}} days)

### Overall

| Period | Charges | Authorized | Declined | Auth Rate | Δ |
|:---|:---|:---|:---|:---|:---|
| **Before** | {{}} | {{}} | {{}} | **{{}}%** | — |
| **Since** | {{}} | {{}} | {{}} | **{{}}%** | **{{}}pp** |

### By 3DS

| Segment | Before | Since | Δ |
|:---|:---|:---|:---|
| Without 3DS | {{}}% (N) | **{{}}%** (N) | {{}}pp |
| With 3DS | {{}}% (N) | **{{}}%** (N) | {{}}pp |

<!-- Repeat for: By 3DS Result, By Geography, By Card Brand, By Instrument, By AVS Zip, Top Decline Reasons -->

### Summary

The overall auth rate **{{improved/declined}} {{X}}pp** since {{SINCE_START}}, primarily driven by:
1. {{top driver}}
2. {{second driver}}
3. {{third driver}}

But **{{key concern}}** remains a drag.

---

# 1. CURRENT STATE SNAPSHOT

## 1.1 Overall: {{X}}% Auth Rate

| Metric | Value |
|:---|:---|
| **Total charges** | {{N}} |
| **Authorized** | {{N}} ({{X}}%) |
| **Issuer declined** | {{N}} ({{X}}%) |

## 1.2 Multi-Dimensional Breakdown

| Dimension | Segment | Charges | Auth Rate | Gap vs Overall |
|:---|:---|:---|:---|:---|
| **3DS** | Without 3DS | N (X%) | **X%** | +Xpp |
| | With 3DS | N (X%) | **X%** | -Xpp |
| **Geography** | Domestic | N (X%) | **X%** | +Xpp |
| | Cross-border | N (X%) | **X%** | -Xpp |
| **Card Brand** | Visa | N (X%) | **X%** | +Xpp |
| | Mastercard | N (X%) | **X%** | -Xpp |
| **Instrument** | Credit | N (X%) | **X%** | +Xpp |
| | Debit | N (X%) | **X%** | -Xpp |
| | Prepaid | N (X%) | **X%** | -Xpp |
| **AVS Zip** | Pass | N (X%) | **X%** | +Xpp |
| | Fail | N (X%) | **X%** | -Xpp |
| | Unchecked | N (X%) | **X%** | -Xpp |

## 1.3 Decline Reasons (N total declines)

| Decline Reason | Count | % of Declines |
|:---|:---|:---|
| `do_not_honor` | N | **X%** |
| `insufficient_funds` | N | **X%** |
<!-- ... top 10 reasons ... -->

**Top 3 decline buckets account for X% of all declines:**
- {{reason1}}: N (X%)
- {{reason2}}: N (X%)
- {{reason3}}: N (X%)

---

# 2. 3D SECURE DEEP-DIVE
<!-- Include if merchant uses 3DS -->

## 2.1 The Core Problem

> **Integration context:** Describe the merchant's 3DS setup here.

| 3DS Status | Charges | Authorized | Declined | Auth Rate |
|:---|:---|:---|:---|:---|
| **Without 3DS** | N | N | N | **X%** |
| **With 3DS** | N | N | N | **X%** |
| **Gap** | | | | **-Xpp** |

### 3DS Result Breakdown

| 3DS Result | Charges | Auth Rate | Meaning |
|:---|:---|:---|:---|
| `authenticated` | N (X%) | **X%** | Full 3DS auth completed |
| `attempt_acknowledged` | N (X%) | **X%** | Attempted but not fully authenticated |

## 2.2 3DS Impact by Domestic vs Cross-Border × Card Brand
<!-- Cross-tab table from Query 2 -->

## 2.3 3DS Impact by Transaction Amount
<!-- From Query 7 -->

## 2.4 Post-3DS-Authentication Decline Analysis
<!-- From Query 3. Key: post-auth decline rate and dominant decline codes -->

## 2.5 BIN-Level 3DS vs No-3DS Comparison
<!-- From Query 5. The "smoking gun" — same BIN, different auth rates -->

| BIN | Issuer | With 3DS | Auth Rate | Without 3DS | Auth Rate | 3DS Penalty |
|:---|:---|:---|:---|:---|:---|:---|
<!-- Top 5 BINs by gap -->

## 2.6 3DS Recommendations
<!-- Tailored to integration shape -->

---

# 3. AVS DEEP-DIVE

## 3.1 AVS Zip/Postal (What Auth Dash Shows)
<!-- From AVS_ZIP dimension -->

## 3.2 Full Combined Picture (Raw AVS Codes)
<!-- From Query 6 -->

## 3.3 AVS Assessment
<!-- Is AVS a lever for this merchant? Usually not if zip pass >75% -->

## 3.4 AVS Recommendations

---

# 4. OTHER DECLINE DRIVERS

## 4.1 CVC-Related Declines
<!-- incorrect_cvc + invalid_cvc count and % -->

## 4.2 Debit & Prepaid Cards
<!-- Lower auth rates, usually insufficient_funds driven -->

## 4.3 do_not_honor Analysis
<!-- Largest decline category — what can be done? -->

---

# 5. PRIORITIZED ACTION PLAN

| Priority | Action | Expected Uplift | Effort | Who |
|:---|:---|:---|:---|:---|
| **P0** | {{action}} | **+X-Xpp** | Low | {{who}} |
| **P1** | {{action}} | **+X-Xpp** | Low-Med | {{who}} |
<!-- ... -->
| | **Total estimated uplift** | **+X-Xpp** | | |
| | **Target auth rate** | **~X-X%** | | |

---

# 6. IMPLEMENTATION GUIDE
<!-- Code samples tailored to integration shape -->
<!-- For external 3DS import: domestic vs cross-border logic, ECI value reference, checklist -->
<!-- For standard integration: Adaptive Acceptance enablement, Radar rule tuning -->

---

# 7. HUBBLE QUERY LINKS

| Query | Link |
|:---|:---|
| Multi-dimensional breakdown | [Hubble]({{url}}) |
<!-- ... one row per query executed ... -->

# 8. INTERNAL RESOURCES

| Resource | Link |
|:---|:---|
| Auth Dash 360 | [go/authdash360](https://go/authdash360) |
| AuthOpt Workbook | [go/one-click](https://go/one-click) |
| Authorization FAQs | [Trailhead](https://trailhead.corp.stripe.com/docs/global-payments-performance-team/authorization-faqs) |
| Network Decline Codes | [go/networkdeclinecodes](https://go/networkdeclinecodes) |
| Radar Rules Docs | [docs.stripe.com](https://docs.stripe.com/radar/rules) |
| Payments Performance (questions) | #payments-performance-ask |
```

---

## Report Writing Guidelines

1. **Lead with the number.** Every section starts with the key metric, not context.
2. **Show before/since deltas** with ✅ (improved) and ❌ (worsened) emoji.
3. **Bold the auth rate** in every table cell for scannability.
4. **Explain "so what"** after each table — don't just present data.
5. **Code samples** in the implementation guide should be copy-pasteable and tailored to the merchant's language/framework when known.
6. **Hubble links** for every query so the reader can verify and re-run.
7. **ECI value reference** is critical for external 3DS import merchants — Visa (05/06) ≠ MC (02/01).
