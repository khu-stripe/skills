---
name: "auth-rate-analysis"
description: "Analyze and optimize payment authorization rates for a Stripe merchant. Run multi-dimensional diagnostics against fact.payments_view in Hubble covering 3DS, AVS, geography, card brand, instrument, and decline reasons. Deep-dive into imported 3DS quality, BIN-level comparison, and post-auth declines. Produce a structured markdown report with prioritized recommendations. Use when asked to: analyze auth rate for a merchant, debug declines, investigate authorization performance, optimize auth, compare auth before/after a date, review 3DS or AVS impact, produce an auth optimization report. Triggers on: 'auth rate', 'authorization rate', 'auth optimization', 'payment declines', 'decline analysis', 'why is auth rate low', 'auth rate dropped', '3DS impact', 'AVS impact', 'decline reasons for acct_', 'auth rate report', 'issuer declines', 'do_not_honor', 'auth rate for acct_'."
allowed-tools: "run_hubble_query get_data_catalog_dataset hubert_data_discovery search_data_catalog_datasets"
---

# Auth Rate Analysis

Systematic authorization rate diagnostic for any Stripe merchant. Produces a multi-dimensional report with actionable, prioritized recommendations.

## Inputs

| Parameter | Required | Description |
|-----------|----------|-------------|
| `merchant_id` | ✅ | Stripe account ID (`acct_*`) |
| `analysis_start` | ✅ | Start date for the "since" period (e.g. `2026-04-15`) |
| `analysis_end` | Optional | End date (defaults to today) |
| `baseline_days` | Optional | Days before `analysis_start` for baseline comparison (default: 30) |

## Core Workflow

```
1. PROFILE     → Identify merchant integration shape (3DS, Stripe.js, Radar, FPAN/token, etc.)
2. BASELINE    → Run before/since comparison query across all dimensions
3. DIAGNOSE    → Run focused deep-dive queries on problem dimensions
4. REPORT      → Write structured markdown report to workspace/
5. RECOMMEND   → Generate prioritized action plan with estimated uplift
```

### Step 1: Profile the Merchant Integration

Run the **Integration Profile** query from [references/queries.md](references/queries.md) (Query 0). This reveals:
- Stripe.js usage, 3DS version/flow/result, payment method type
- Whether 3DS is Stripe-native or externally imported (no `authentication_flow` = imported)
- Radar/Shield usage, saved card patterns
- Integration surface (dashboard, API, checkout, etc.)

**Key integration shapes** that change the analysis strategy:
| Shape | Implications |
|-------|-------------|
| FPAN + No Stripe.js + External 3DS import | 3DS quality is #1 suspect; CVC validation gaps likely |
| Stripe.js + Radar + Stripe 3DS | Standard flow; focus on Radar rules, 3DS exemption strategy |
| Token/saved card + MIT | Focus on network token adoption, credential-on-file flags |
| Connect platform (OBO) | Check platform vs connected account country, settlement merchant |

### Step 2: Run Before/Since Comparison

Run **Query 1** (Before vs Since) from [references/queries.md](references/queries.md). This single UNION ALL query produces comparison rows across 7 dimensions:
- Overall, 3DS status, 3DS result type, Geography, Card Brand, Instrument, AVS Zip

**Date math:**
- "Before" period: `analysis_start - baseline_days` to `analysis_start`
- "Since" period: `analysis_start` to `analysis_end`

### Step 3: Run Focused Deep-Dives

Based on Step 2 results, run the relevant deep-dive queries from [references/queries.md](references/queries.md):

| Signal from Step 2 | Deep-Dive Query |
|---------------------|-----------------|
| 3DS auth rate < no-3DS auth rate by >5pp | Query 2 (3DS × Geo × Brand), Query 5 (BIN-level 3DS comparison) |
| `authenticated` < `attempt_acknowledged` | Query 3 (Post-auth decline reasons) — likely cryptogram/ECI quality issue |
| High `do_not_honor` share (>25%) | Query 4 (Decline reasons with raw codes) |
| AVS fail/unchecked auth rate < pass by >15pp | Query 6 (AVS raw code breakdown) |
| Cross-border auth rate diverging from domestic | Query 2 with geographic focus |
| Debit/prepaid auth rate lagging credit by >10pp | Check `insufficient_funds` concentration on debit |

### Step 4: Write the Report

Use the template in [references/report-template.md](references/report-template.md). Write to `workspace/auth_optimization_{merchant_id}.md`.

**Report sections:**
0. Before vs Since quick comparison (if baseline exists)
1. Current state snapshot (overall + multi-dimensional breakdown + decline reasons)
2. 3DS deep-dive (if applicable: core problem, geo×brand cross-tab, amount buckets, post-auth declines, BIN-level smoking gun)
3. AVS deep-dive (zip vs address, raw AVS codes, combined picture)
4. Other decline drivers (CVC, debit/prepaid, `do_not_honor`)
5. Prioritized action plan with estimated uplift
6. Implementation guide (tailored to integration shape)
7. Hubble query links

### Step 5: Recommendations Framework

Prioritize recommendations by expected auth rate uplift × implementation effort:

| Priority | Criteria |
|----------|----------|
| **P0** | >2pp uplift, low effort, clear root cause |
| **P1** | 1-2pp uplift, or diagnostic needed first |
| **P2** | <1pp uplift, or medium-high effort |
| **P3** | Marginal uplift, optional |

**Common recommendation patterns by integration shape:**

**External 3DS import (FPAN, server-side API):**
- P0: Stop importing 3DS for domestic US (not legally required, often hurts auth)
- P0: Audit cryptogram/ECI/dsTransID payload quality
- P0: Verify ECI values per card brand (Visa 05/06 ≠ MC 02/01)
- P1: Check cryptogram freshness (>15 min = likely stale)

**Standard Stripe.js integration:**
- P1: Enable Adaptive Acceptance
- P1: Optimize Radar rules (reduce false positive blocks)
- P2: Consider network tokens for saved cards
- P2: Implement Optimized Checkout Suite

**General (all integrations):**
- P1: Add client-side CVC validation if missing
- P2: Improve billing address collection for AVS
- P3: Customer-facing decline messaging improvements

## Key Field Reference

See [references/field-reference.md](references/field-reference.md) for complete field definitions, gotchas, and the auth rate calculation formula.

## SQL Query Templates

See [references/queries.md](references/queries.md) for all parameterized SQL templates used in this analysis.

## Report Template

See [references/report-template.md](references/report-template.md) for the full markdown report structure.
