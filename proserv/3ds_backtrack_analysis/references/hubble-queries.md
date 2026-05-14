# Hubble Query Templates for 3DS Analysis

All queries target `events.shield_rules_execution_details`. The key
challenge is extracting multiple attributes from the `all_rule_attributes`
JSON array, which requires multiple CROSS JOIN UNNEST operations.

Replace these placeholders before running:
- `{{ACCOUNT_ID}}` — merchant account ID (e.g., `acct_1SmQXxQOv4Ao6J0Y`)
- `{{START_DAY}}` — start date in `YYYYMMDD00` format (e.g., `2026032600`)
- `{{END_DAY}}` — end date in `YYYYMMDD00` format (e.g., `2026051500`)

## Common WHERE Clause

All queries share this base filter:

```sql
WHERE merchant_id = '{{ACCOUNT_ID}}'
  AND livemode = true
  AND day >= '{{START_DAY}}'
  AND day < '{{END_DAY}}'
  AND shield_op = 'EVALUATE_RADAR_PRE_AUTHZ'
```

## Common UNNEST Pattern

The `all_rule_attributes` column is a JSON array. Each element has:
- `$.definition.name` — attribute name
- `$.string_value` — string attributes
- `$.numeric_value` — numeric attributes
- `$.boolean_value` — boolean attributes

To extract multiple attributes, use separate CROSS JOIN UNNEST aliases
and filter each by its `definition.name`:

```sql
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t(attr)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t2(attr2)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t3(attr3)
```

Then in the WHERE clause:
```sql
AND json_extract_scalar(attr, '$.definition.name') = 'card_3d_secure_support'
AND json_extract_scalar(attr2, '$.definition.name') = 'card_3d_secure_result'
AND json_extract_scalar(attr3, '$.definition.name') = 'risk_score'
```

## Query 1: 3DS Classification Overview

High-level counts by 3DS support type and result.

```sql
SELECT
  json_extract_scalar(attr, '$.string_value') AS card_3d_secure_support,
  json_extract_scalar(attr2, '$.string_value') AS card_3d_secure_result,
  count(*) AS txn_count
FROM events.shield_rules_execution_details
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t(attr)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t2(attr2)
WHERE merchant_id = '{{ACCOUNT_ID}}'
  AND livemode = true
  AND day >= '{{START_DAY}}'
  AND day < '{{END_DAY}}'
  AND shield_op = 'EVALUATE_RADAR_PRE_AUTHZ'
  AND json_extract_scalar(attr, '$.definition.name') = 'card_3d_secure_support'
  AND json_extract_scalar(attr2, '$.definition.name') = 'card_3d_secure_result'
GROUP BY 1, 2
ORDER BY 1, 2
```

### Key field values

| `card_3d_secure_support` | Meaning |
|---|---|
| `recommended` | Stripe recommends 3DS |
| `optional` | 3DS available but not recommended |
| `not_supported` | Card does not support 3DS |

| `card_3d_secure_result` | Meaning |
|---|---|
| `authenticated` | 3DS completed, cardholder verified |
| `attempt_acknowledged` | 3DS attempted, issuer acknowledged |
| `not_attempted` | No 3DS was performed |

## Query 2: Per-Risk-Score Breakdown with 3DS Status

Granular data for building risk distribution charts and computing
threshold scenarios. Uses `is_3ds_import` to determine if the merchant
actually triggered 3DS (vs Stripe's recommendation).

```sql
SELECT
  json_extract_scalar(attr, '$.string_value') AS card_3d_secure_support,
  CAST(FLOOR(CAST(json_extract_scalar(attr3, '$.numeric_value') AS double)) AS integer) AS risk_score,
  CASE
    WHEN json_extract_scalar(attr2, '$.string_value') = 'not_attempted'
    THEN 'no_3ds'
    ELSE '3ds_done'
  END AS threeds_status,
  count(*) AS txn_count
FROM events.shield_rules_execution_details
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t(attr)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t2(attr2)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t3(attr3)
WHERE merchant_id = '{{ACCOUNT_ID}}'
  AND livemode = true
  AND day >= '{{START_DAY}}'
  AND day < '{{END_DAY}}'
  AND shield_op = 'EVALUATE_RADAR_PRE_AUTHZ'
  AND json_extract_scalar(attr, '$.definition.name') = 'card_3d_secure_support'
  AND json_extract_scalar(attr2, '$.definition.name') = 'card_3d_secure_result'
  AND json_extract_scalar(attr3, '$.definition.name') = 'risk_score'
GROUP BY 1, 2, 3
ORDER BY 1, 2, 3
```

### Alternative: Using is_3ds_import

If the merchant uses standalone 3DS import (sends 3DS results separately
from Stripe's managed flow), use `is_3ds_import` instead of
`card_3d_secure_result`:

```sql
SELECT
  json_extract_scalar(attr, '$.string_value') AS card_3d_secure_support,
  CAST(FLOOR(CAST(json_extract_scalar(attr3, '$.numeric_value') AS double)) AS integer) AS risk_score,
  json_extract_scalar(attr4, '$.boolean_value') AS is_3ds_import,
  count(*) AS txn_count
FROM events.shield_rules_execution_details
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t(attr)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t3(attr3)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t4(attr4)
WHERE merchant_id = '{{ACCOUNT_ID}}'
  AND livemode = true
  AND day >= '{{START_DAY}}'
  AND day < '{{END_DAY}}'
  AND shield_op = 'EVALUATE_RADAR_PRE_AUTHZ'
  AND json_extract_scalar(attr, '$.definition.name') = 'card_3d_secure_support'
  AND json_extract_scalar(attr3, '$.definition.name') = 'risk_score'
  AND json_extract_scalar(attr4, '$.definition.name') = 'is_3ds_import'
GROUP BY 1, 2, 3
ORDER BY 1, 2, 3
```

## Query 3: Risk Score Distribution Statistics

Summary stats to confirm the recommended/optional boundary.

```sql
SELECT
  json_extract_scalar(attr, '$.string_value') AS card_3d_secure_support,
  MIN(CAST(json_extract_scalar(attr3, '$.numeric_value') AS double)) AS min_risk,
  MAX(CAST(json_extract_scalar(attr3, '$.numeric_value') AS double)) AS max_risk,
  APPROX_PERCENTILE(CAST(json_extract_scalar(attr3, '$.numeric_value') AS double), 0.5) AS median_risk,
  APPROX_PERCENTILE(CAST(json_extract_scalar(attr3, '$.numeric_value') AS double), 0.9) AS p90_risk,
  APPROX_PERCENTILE(CAST(json_extract_scalar(attr3, '$.numeric_value') AS double), 0.95) AS p95_risk,
  count(*) AS txn_count
FROM events.shield_rules_execution_details
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t(attr)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t3(attr3)
WHERE merchant_id = '{{ACCOUNT_ID}}'
  AND livemode = true
  AND day >= '{{START_DAY}}'
  AND day < '{{END_DAY}}'
  AND shield_op = 'EVALUATE_RADAR_PRE_AUTHZ'
  AND json_extract_scalar(attr, '$.definition.name') = 'card_3d_secure_support'
  AND json_extract_scalar(attr3, '$.definition.name') = 'risk_score'
GROUP BY 1
ORDER BY 1
```

## Query 4: Disputed Payment Details

Join Shield data with specific payment intents that have disputes. First
identify disputed PIs from the disputes table, then query Shield for
their risk attributes.

```sql
SELECT
  payment_intent_id,
  json_extract_scalar(attr, '$.string_value') AS card_3d_secure_support,
  json_extract_scalar(attr2, '$.string_value') AS card_3d_secure_result,
  CAST(json_extract_scalar(attr3, '$.numeric_value') AS double) AS risk_score,
  json_extract_scalar(attr4, '$.boolean_value') AS is_3ds_import,
  json_extract_scalar(attr5, '$.string_value') AS receipt_email,
  json_extract_scalar(attr6, '$.string_value') AS ip_country
FROM events.shield_rules_execution_details
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t(attr)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t2(attr2)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t3(attr3)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t4(attr4)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t5(attr5)
CROSS JOIN UNNEST(CAST(json_parse(all_rule_attributes) AS array(json))) AS t6(attr6)
WHERE merchant_id = '{{ACCOUNT_ID}}'
  AND livemode = true
  AND day >= '{{START_DAY}}'
  AND day < '{{END_DAY}}'
  AND shield_op = 'EVALUATE_RADAR_PRE_AUTHZ'
  AND payment_intent_id IN (
    -- Replace with actual disputed PI IDs
    'pi_xxx', 'pi_yyy', 'pi_zzz'
  )
  AND json_extract_scalar(attr, '$.definition.name') = 'card_3d_secure_support'
  AND json_extract_scalar(attr2, '$.definition.name') = 'card_3d_secure_result'
  AND json_extract_scalar(attr3, '$.definition.name') = 'risk_score'
  AND json_extract_scalar(attr4, '$.definition.name') = 'is_3ds_import'
  AND json_extract_scalar(attr5, '$.definition.name') = 'receipt_email'
  AND json_extract_scalar(attr6, '$.definition.name') = 'ip_country'
```

### Finding Disputed Payment Intents

To get the list of disputed PIs, query the disputes table first:

```sql
SELECT
  d.payment_intent_id,
  d.amount,
  d.currency,
  d.reason,
  d.status,
  d.created
FROM mart.fct_disputes d
WHERE d.merchant_id = '{{ACCOUNT_ID}}'
  AND d.created >= {{START_UNIX_TIMESTAMP}}
  AND d.created < {{END_UNIX_TIMESTAMP}}
ORDER BY d.created
```

## Query 5: Early Fraud Warning Details

Similar to disputes, query EFW payments against Shield data.

```sql
SELECT
  efw.payment_intent_id,
  efw.amount,
  efw.currency,
  efw.fraud_type,
  efw.created
FROM mart.fct_early_fraud_warnings efw
WHERE efw.merchant_id = '{{ACCOUNT_ID}}'
  AND efw.created >= {{START_UNIX_TIMESTAMP}}
  AND efw.created < {{END_UNIX_TIMESTAMP}}
ORDER BY efw.created
```

Then join the resulting PI IDs back with the Shield query (Query 4
pattern) to get risk scores and 3DS status.

## Performance Notes

- Triple CROSS JOIN UNNEST is expensive. For large merchants (>100k
  transactions), consider pre-filtering or querying in smaller time
  windows.
- The `day` partition column uses `YYYYMMDD00` format (note the trailing
  `00`).
- Always include `shield_op = 'EVALUATE_RADAR_PRE_AUTHZ'` to get only
  the pre-authorization evaluation, not post-auth or other Shield ops.
- For merchants with very high volume, consider querying the risk score
  distribution separately from the 3DS classification to avoid the
  combinatorial explosion of multiple UNNESTs.
