---
name: 3ds-analysis
description: >-
  Analyze a merchant's 3DS triggering strategy by comparing their current
  behavior against Stripe's built-in Radar 3DS rules across multiple risk
  score thresholds. Produces scenario comparisons, risk distribution charts,
  dispute/EFW coverage matrices, and actionable recommendations. Use this
  skill whenever the engineer asks to analyze 3DS rates, compare 3DS
  scenarios, review 3DS triggering, audit 3DS coverage, investigate
  disputes or early fraud warnings related to 3DS, or evaluate whether a
  merchant should adjust their 3DS threshold. Also trigger for "3DS
  analysis", "3DS optimization", "Radar 3DS review", "3DS trigger rate",
  or "fraud/dispute coverage analysis".
---

# 3DS Multi-Scenario Analysis

Compares a merchant's actual 3DS triggering behavior against hypothetical
scenarios based on Stripe's Radar risk scores and 3DS classification
(recommended vs optional). The output is a data-driven report showing
which threshold configuration would cover observed disputes and Early
Fraud Warnings, with the tradeoff of increased payment friction.

## Prerequisites

- Merchant account ID (`acct_...`)
- Analysis time period (start and end dates)
- Access to Hubble for querying `events.shield_rules_execution_details`
  and dispute/EFW tables

## Procedure

### Step 1: Collect Data

Run the Hubble queries from `references/hubble-queries.md`. There are
four queries to execute:

1. **3DS classification counts** — total transactions grouped by
   `card_3d_secure_result` (recommended vs attempt_acknowledged/optional)
   and whether 3DS was actually performed (`is_3ds_import`)
2. **Per-risk-score breakdown** — for optional transactions, count
   `is_3ds_import` true/false per `risk_score`
3. **Disputed payments** — all disputes in the period, joined with Shield
   data to get risk score and 3DS status
4. **Early Fraud Warnings** — all EFWs, joined with Shield data for risk
   score, 3DS status, and fraud type

Read `references/hubble-queries.md` for the full SQL templates. Replace
the placeholders `{{ACCOUNT_ID}}` and `{{START_DATE}}`/`{{END_DATE}}`
with the merchant's values.

### Step 2: Understand the Data Model

Stripe's Radar classifies each card transaction's 3DS recommendation:

| `card_3d_secure_result` | Meaning | Risk score range |
|---|---|---|
| `recommended` | Stripe recommends 3DS for this card | Typically 30+ |
| `attempt_acknowledged` | 3DS is available but not recommended (optional) | Typically 0 to 29 |

The `is_3ds_import` field indicates whether the merchant actually
performed 3DS authentication (via standalone 3DS import or Stripe-managed
3DS). This is distinct from the Radar recommendation.

The boundary between optional and recommended is not fixed at exactly 30
for all merchants. Always verify with the query data where the actual
cutoff falls.

### Step 3: Calculate Scenarios

Build these scenario comparisons:

| Scenario | Eligible transactions |
|---|---|
| A: Current | Merchant's actual 3DS-triggered count |
| B: Recommended only | All `card_3d_secure_result = recommended` |
| Rec + Opt > 25 | B + optional transactions with risk > 25 |
| Rec + Opt > 20 | B + optional transactions with risk > 20 |
| Rec + Opt > 15 | B + optional transactions with risk > 15 |
| Rec + Opt > 10 | B + optional transactions with risk > 10 |

For each scenario compute:
- **Eligible transactions**: count of transactions that would trigger 3DS
- **3DS rate**: eligible / total
- **Delta vs current**: difference in eligible count and percentage points

### Step 4: Build Risk Distribution

Group optional transactions by individual risk score (0 through max) with
counts of 3DS-done vs no-3DS. Also aggregate into bands (0-4, 5-9,
10-14, 15-19, 20-24, 25-29) for a summary table.

Do the same for recommended transactions (30+), grouping by score and
computing 5-score bands.

### Step 5: Dispute and EFW Coverage Matrix

For each disputed payment and EFW, determine which scenarios would have
covered it:

- Look up its risk score
- Check whether `card_3d_secure_result` is recommended or optional
- For each threshold scenario, mark Yes/No based on whether the risk
  score exceeds that threshold

Flag any payments that fall below all thresholds — these represent the
fundamental gap where no reasonable threshold would catch them without
excessive friction.

### Step 6: Analyze Radar Input Quality

Check whether the merchant passes key Radar signals:
- `receipt_email` / email
- IP address
- Billing address

For disputed and EFW transactions specifically, note which signals were
missing. Missing signals significantly reduce Radar's ability to
differentiate risk, which often manifests as low risk scores (10-15) for
transactions that later turn out to be fraudulent.

### Step 7: Generate Output

The analysis can be presented in three formats:

#### Canvas (interactive, for internal review)

Build a Cursor Canvas (`.canvas.tsx`) with:
- KPI stats row (total txns, recommended count, current 3DS count, rates)
- Scenario comparison table
- Stacked bar charts for risk distribution (optional and recommended)
- Risk band summary tables
- Dispute and EFW coverage matrices
- Callouts for key findings
- Summary with recommendations

Read `references/html-template.md` for the HTML structure that maps to
each canvas section.

#### HTML (shareable, Stripe-branded)

Export to a standalone HTML file with Stripe's corporate color palette.
Read `references/html-template.md` for the full template including CSS
variables, Chart.js integration, and print styles.

Key design elements:
- Stripe color palette: navy (#0A2540), purple (#635BFF), red (#FF4F64),
  orange (#FF7A00), green (#00D4AA)
- Light background (#F6F9FC), white cards/tables
- Table headers in navy with white text, uppercase letter-spacing
- Callouts with colored left border
- Chart.js stacked bar charts with Stripe purple/red
- Footer: "Stripe Professional Services | Prepared for {Merchant}"

#### PDF (formal deliverable)

Use ReportLab to generate a styled PDF. For Chinese content, register
macOS CJK fonts (`/System/Library/Fonts/STHeiti Medium.ttc`). Use
matplotlib for embedded charts.

### Step 8: Bilingual Support

If the merchant requires Chinese output, produce both English and Chinese
versions. Translate all labels, headers, callout text, and summary
paragraphs. Keep technical terms (Radar, 3DS, PaymentIntent, risk score)
in English for clarity.

## Key Insights to Surface

Every analysis should evaluate and comment on these points:

1. **Recommended vs Optional boundary**: where does the cutoff actually
   fall? Is Opt > 30 always identical to Recommended-only?
2. **Dispute coverage gap**: which threshold would have caught the
   disputes? What is the friction cost?
3. **EFW coverage**: are there EFW payments below all thresholds?
4. **Radar signal quality**: does the merchant pass email/IP? If not,
   this is the primary recommendation over adjusting thresholds.
5. **First-time cards**: are flagged transactions using cards with no
   Stripe history? This compounds the missing-signals problem.

## Completion Criteria

- All four Hubble queries executed and data validated
- At least 5 scenario comparisons calculated (current, recommended,
  and 3+ optional thresholds)
- Every dispute and EFW mapped against all scenarios
- Risk distribution visualized (charts + tables)
- Radar input quality assessed
- Output generated in requested format (canvas/HTML/PDF)
- Summary includes actionable recommendations
