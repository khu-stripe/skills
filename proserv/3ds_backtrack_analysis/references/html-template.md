# HTML Export Template

Stripe-branded standalone HTML template for the 3DS analysis report.
Uses Chart.js for stacked bar charts. Self-contained except for the
Chart.js CDN dependency.

## Stripe Color Palette

```css
:root {
  --stripe-purple: #635BFF;     /* Primary accent */
  --stripe-purple-light: #7A73FF;
  --stripe-navy: #0A2540;       /* Primary text, table headers */
  --stripe-slate: #425466;      /* Body text */
  --stripe-gray: #8898AA;       /* Secondary text, labels */
  --stripe-light: #F6F9FC;      /* Page background */
  --stripe-border: #E3E8EE;     /* Borders */
  --stripe-white: #FFFFFF;      /* Card/table backgrounds */
  --stripe-green: #00D4AA;      /* Success */
  --stripe-green-bg: #E6FAF5;
  --stripe-orange: #FF7A00;     /* Warning */
  --stripe-orange-bg: #FFF3E6;
  --stripe-red: #FF4F64;        /* Danger */
  --stripe-red-bg: #FFE8EB;
  --stripe-blue: #00A2E8;       /* Info */
  --stripe-blue-bg: #E6F6FC;
  --stripe-purple-bg: #F0EFFF;  /* Highlighted rows */
}
```

## Document Structure

```html
<!DOCTYPE html>
<html lang="{{LANG}}">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{MERCHANT}} 3DS Analysis</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>
<style>/* CSS from palette above + component styles */</style>
</head>
<body>
<div class="container">

  <!-- Title -->
  <h1>{{TITLE}}</h1>
  <p class="subtitle">{{DATE_RANGE}} | {{ACCOUNT_ID}} | {{TOTAL}} transactions</p>

  <!-- KPI Grid -->
  <div class="kpi-grid">...</div>

  <!-- Scenario Comparison Table -->
  <!-- Extended Thresholds Table -->
  <!-- Risk Distribution Charts (Chart.js) -->
  <!-- Risk Band Summary Tables (two-column) -->
  <!-- Disputed Payments Coverage Matrix -->
  <!-- EFW Coverage Matrix -->
  <!-- Summary + Callouts -->

  <!-- Footer -->
  <div class="footer">...</div>

</div>
<script>/* Chart.js initialization */</script>
</body>
</html>
```

## Component Styles

### KPI Cards

```css
.kpi-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 12px;
  margin-bottom: 14px;
}
.kpi {
  background: var(--stripe-white);
  border: 1px solid var(--stripe-border);
  border-radius: 8px;
  padding: 16px 12px;
  text-align: center;
}
.kpi .value {
  font-size: 1.6rem;
  font-weight: 700;
  color: var(--stripe-navy);
}
.kpi .label {
  font-size: 0.72rem;
  color: var(--stripe-gray);
  margin-top: 4px;
  text-transform: uppercase;
  letter-spacing: 0.03em;
}
/* Accent variants: colored left border + value color */
.kpi.purple { border-left: 3px solid var(--stripe-purple); }
.kpi.purple .value { color: var(--stripe-purple); }
.kpi.orange { border-left: 3px solid var(--stripe-orange); }
.kpi.orange .value { color: var(--stripe-orange); }
.kpi.blue { border-left: 3px solid var(--stripe-blue); }
.kpi.blue .value { color: var(--stripe-blue); }
```

### Tables

```css
table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
  font-size: 0.82rem;
  background: var(--stripe-white);
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid var(--stripe-border);
}
th {
  background: var(--stripe-navy);
  color: var(--stripe-white);
  padding: 10px 12px;
  font-weight: 600;
  font-size: 0.78rem;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}
td {
  padding: 8px 12px;
  border-bottom: 1px solid var(--stripe-border);
  color: var(--stripe-slate);
}
/* Row highlights */
tr.purple td { background: var(--stripe-purple-bg); }
tr.orange td { background: var(--stripe-orange-bg); }
tr.red td { background: var(--stripe-red-bg); }
```

### Callouts

```css
.callout {
  border-radius: 8px;
  padding: 16px 18px;
  margin: 14px 0;
  border-left: 4px solid;
  background: var(--stripe-white);
}
.callout.purple { background: var(--stripe-purple-bg); border-color: var(--stripe-purple); }
.callout.orange { background: var(--stripe-orange-bg); border-color: var(--stripe-orange); }
.callout.red { background: var(--stripe-red-bg); border-color: var(--stripe-red); }
```

### Two-Column Layout

```css
.two-col { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
```

### Section Headings

```css
h2 {
  font-size: 1.25rem;
  font-weight: 600;
  color: var(--stripe-navy);
  padding-bottom: 8px;
  border-bottom: 2px solid var(--stripe-purple);
  display: inline-block;
}
/* Wrap in .h2-wrap for proper margins */
.h2-wrap { margin: 28px 0 12px; }
```

## Chart.js Configuration

Use Chart.js v4 for stacked bar charts. Stripe brand colors for
datasets:

```javascript
Chart.defaults.font.family = "-apple-system, 'PingFang SC', 'Helvetica Neue', sans-serif";
Chart.defaults.color = '#425466';

new Chart(document.getElementById('chartId'), {
  type: 'bar',
  data: {
    labels: ['0','1','2',...],  // risk scores
    datasets: [
      {
        label: '3DS Triggered',
        data: [...],
        backgroundColor: '#635BFFcc',  // Stripe purple with transparency
        borderRadius: 2,
        stack: 's'
      },
      {
        label: 'No 3DS',
        data: [...],
        backgroundColor: '#FF4F64cc',  // Stripe red with transparency
        borderRadius: 2,
        stack: 's'
      }
    ]
  },
  options: {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: {
        position: 'top',
        labels: { font: { size: 11 }, usePointStyle: true, pointStyle: 'rectRounded', padding: 16 }
      }
    },
    scales: {
      x: { grid: { display: false }, ticks: { font: { size: 10 } } },
      y: { beginAtZero: true, grid: { color: '#E3E8EE' }, ticks: { font: { size: 10 } } }
    }
  }
});
```

## Footer

```html
<div class="footer">
  <div class="footer-left">
    <span class="footer-text">Stripe Professional Services | Prepared for {{MERCHANT}}</span>
  </div>
  <div class="footer-text">{{MONTH}} {{YEAR}}</div>
</div>
```

```css
.footer {
  margin-top: 40px;
  padding-top: 20px;
  border-top: 1px solid var(--stripe-border);
  display: flex;
  justify-content: space-between;
}
.footer-text {
  font-size: 0.72rem;
  color: var(--stripe-gray);
}
```

## Print Styles

Include print-friendly overrides for PDF export via browser print:

```css
@media print {
  body { background: #fff; }
  .container { max-width: 100%; padding: 16px; }
  .chart-wrap { break-inside: avoid; }
  table { break-inside: avoid; }
  th, tr.red td, tr.purple td, tr.orange td,
  .kpi.purple, .kpi.orange, .kpi.blue, .callout {
    print-color-adjust: exact;
    -webkit-print-color-adjust: exact;
  }
}
```

## Chinese Language Support

For Chinese reports, set `lang="zh-CN"` and use the font stack:

```css
font-family: -apple-system, "PingFang SC", "Microsoft YaHei", "Helvetica Neue", sans-serif;
```

Translate all UI labels but keep technical terms in English: Radar,
3DS, PaymentIntent, risk score, Recommended, Optional.
