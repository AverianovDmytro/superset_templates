---
name: superset-handlebars-table
description: >
  Use this skill whenever the user wants to create a styled metrics or data table
  in Apache Superset using the Handlebars chart type. Triggers include: "create a
  Handlebars table in Superset", "metrics table in Superset", "Superset custom
  table with CSS", "styled table chart Superset", or any request to show tabular
  data with custom formatting, color coding, or KPI comparisons in Superset.
  Also use when the user mentions plan vs. actual, Abweichung, or color-coded
  cells in a Superset dashboard.
---

# Superset Handlebars Table

This skill guides you through building a fully styled, production-ready metrics
table in Apache Superset using the Handlebars chart type — from SQL Virtual
Dataset to the final template.

---

## Critical Rules (learned from production)

These rules are non-negotiable. Violating any of them causes the output to
render as raw HTML text in the browser.

### 1. Never use `<table>` elements
Superset's client-side DOMPurify sanitizer **strips** `<table>`, `<thead>`,
`<tbody>`, `<tr>`, `<td>`, `<th>`. Use **CSS Grid with divs** instead.

### 2. Never use HTML comments
`<!-- comment -->` breaks the sanitizer's HTML parser. Everything after the
comment may render as raw text. Remove all comments from the template.

### 3. Never use inline `style="..."` attributes on container divs
The sanitizer strips `style` attributes from divs. When it removes
`<div style="overflow-x:auto;">`, it drops the tag but keeps the inner content
as escaped text. Use **CSS classes** for all styling.

### 4. Put `<style>` inside the Handlebars Template field
The separate "CSS Styles" field in Superset injects CSS as raw text into the
rendered output (a bug in Superset 6.x). Leave that field **empty**.

### 5. Enable `HTML_SANITIZATION = False` in superset_config.py
Without this, Superset's server-side sanitizer strips the `<style>` tag but
leaves its text content visible. Add to `superset_config.py`:

```python
HTML_SANITIZATION = False
```

Then restart Superset:
```bash
TAG=6.1.0 docker compose -f docker-compose-image-tag.yml down
TAG=6.1.0 docker compose -f docker-compose-image-tag.yml up -d
```

### 6. Use bracket notation for Superset column names
Superset renames aggregated columns: `feed_tons` → `SUM(feed_tons)`.
In Handlebars, access them with bracket notation:

```handlebars
{{this.[SUM(feed_tons)]}}
{{this.[AVG(abw_pct)]}}
```

### 7. Use HTML entities for special characters
Arrows, dashes, umlauts — use entities to avoid encoding issues:

| Character | Entity |
|-----------|--------|
| ▲ | `&#9650;` |
| ▼ | `&#9660;` |
| ▶ | `&#9654;` |
| — | `&#8212;` |
| ≥ | `&#8805;` |
| Σ | `&#931;` |
| ü | `&#252;` |
| · | `&#183;` |

### 8. `{{#if}}` only for class names and text — never wrapping HTML elements

This is the most subtle rule. In Superset's Handlebars, `{{#if}}` used as a
block wrapper around `<div>` or `<span>` elements causes those elements to
render as raw HTML text. `{{#unless}}` is also not supported.

**Broken — causes raw HTML output:**
```handlebars
{{#if (eq fisc_month "99 Gesamt")}}
<div class="tbl-total">...</div>
{{else}}
<div class="tbl-row">...</div>
{{/if}}
```

**Correct — `{{#if}}` only inside attributes and text nodes:**
```handlebars
<div class="tbl-row {{#if (eq fisc_month "99 Gesamt")}}tbl-total{{/if}}">
  <span class="{{#if (eq ...)}}total-label{{else}}month{{/if}}">
    {{#if (eq fisc_month "99 Gesamt")}}&#931; Gesamt{{else}}{{fisc_month}}{{/if}}
  </span>
  <span class="arr {{#if (gte a b)}}up{{else}}dn{{/if}}">
    {{#if (gte a b)}}&#9650;{{else}}&#9660;{{/if}}
  </span>
</div>
```

The rule: every `{{#if}}` block must produce **only text** (class names,
entities, values) — never open or close an HTML tag.

---

## Step 1 — Virtual Dataset SQL

Pre-compute all derived columns (plan, deviation) in SQL so the Handlebars
template stays logic-free. Use a Virtual Dataset in Superset
(Datasets → + Dataset → Write SQL query).

### Basic monthly aggregation

```sql
WITH base AS (
    SELECT
          date
        , feed_tons, oil_tons, meal_tons

        , CASE
            WHEN EXTRACT(MONTH FROM date) >= 9
                THEN TO_CHAR(date, 'YY') || '-' || TO_CHAR(date + INTERVAL '1 year', 'YY')
            ELSE
                TO_CHAR(date - INTERVAL '1 year', 'YY') || '-' || TO_CHAR(date, 'YY')
          END AS fisc_year

        , LPAD(
            (CASE
                WHEN EXTRACT(MONTH FROM date) >= 9
                    THEN EXTRACT(MONTH FROM date)::int - 8
                ELSE EXTRACT(MONTH FROM date)::int + 4
             END)::text, 2, '0'
          ) || ' ' || TO_CHAR(date, 'Mon') AS fisc_month

        , DATE_PART('days',
              DATE_TRUNC('month', date) + INTERVAL '1 month' - INTERVAL '1 day'
          ) AS days_in_month

    FROM your_schema.your_table
)
SELECT
      fisc_year
    , fisc_month
    , ROUND(SUM(feed_tons)::numeric, 1)                                       AS feed_tons
    , ROUND(SUM(oil_tons)::numeric,  1)                                       AS oil_tons
    , ROUND(SUM(meal_tons)::numeric, 1)                                       AS meal_tons
    , ROUND((575.0 * MAX(days_in_month))::numeric, 1)                         AS plan_tons
    , ROUND((SUM(feed_tons) - 575.0 * MAX(days_in_month))::numeric, 1)       AS abw_tons
    , ROUND(((SUM(feed_tons) / (575.0 * MAX(days_in_month))) - 1) * 100, 1)  AS abw_pct
FROM base
GROUP BY fisc_year, fisc_month
ORDER BY fisc_year, fisc_month
```

### With totals row (UNION ALL)

To show a Σ Gesamt footer row, add a UNION ALL. The marker `'99 Gesamt'`
sorts last alphabetically. Detect it in the template with
`{{#if (eq fisc_month "99 Gesamt")}}`.

```sql
WITH monthly AS (
    -- ... same as basic query above ...
)

SELECT * FROM monthly

UNION ALL

SELECT
      MAX(fisc_year)                                                      AS fisc_year
    , '99 Gesamt'                                                         AS fisc_month
    , ROUND(SUM(feed_tons)::numeric,  1)                                  AS feed_tons
    , ROUND(SUM(oil_tons)::numeric,   1)                                  AS oil_tons
    , ROUND(SUM(meal_tons)::numeric,  1)                                  AS meal_tons
    , ROUND(SUM(plan_tons)::numeric,  1)                                  AS plan_tons
    , ROUND(SUM(abw_tons)::numeric,   1)                                  AS abw_tons
    , ROUND(((SUM(feed_tons) / NULLIF(SUM(plan_tons), 0)) - 1) * 100, 1) AS abw_pct
FROM monthly

ORDER BY fisc_month
```

**Adjust**: replace `575.0` with actual plan/day, `your_schema.your_table`
with your table, and add/remove metric columns as needed.

---

## Step 2 — Superset Chart Configuration

- Chart type: **Handlebars**
- Dimension: `fisc_month`
- Metrics: `feed_tons` (SUM), `plan_tons` (SUM), `abw_tons` (SUM),
  `abw_pct` (AVG), `oil_tons` (SUM), `meal_tons` (SUM)
- Do **not** add string columns as metrics — they become `COUNT_DISTINCT(col)`
- CSS Styles field: **leave empty**

---

## Step 3 — Handlebars Template

Proven production template (v4) with Σ Gesamt totals row.
All `{{#if}}` blocks are inline only — never wrapping HTML elements.

```handlebars
<style>
.mc{background:#fff;border-radius:12px;box-shadow:0 1px 4px rgba(0,0,0,.06),0 4px 16px rgba(0,0,0,.08);overflow:hidden;max-width:1100px;margin:0 auto;font-family:'Segoe UI',system-ui,sans-serif;font-size:13.5px;color:#1e2430;}
.mc-hdr{padding:20px 28px 16px;border-bottom:1px solid #e8ecf1;display:flex;align-items:center;gap:14px;}
.mc-title{font-size:17px;font-weight:600;letter-spacing:-0.2px;}
.mc-badge{display:inline-flex;align-items:center;gap:5px;background:#eef2ff;color:#4361ee;font-size:12px;font-weight:600;padding:3px 10px;border-radius:20px;}
.tbl-wrap{overflow-x:auto;}
.tbl-head,.tbl-row{display:grid;grid-template-columns:120px 1fr 1fr 1fr 1fr 1fr 1fr;}
.tbl-head{background:#f8f9fb;border-bottom:2px solid #e8ecf1;}
.tbl-row{border-bottom:1px solid #f0f2f5;}
.tbl-row:hover .c{background:#fafbfc;}
.tbl-row.tbl-total{border-top:2px solid #c7d2fe;background:#eef2ff;border-bottom:none;}
.tbl-row.tbl-total:hover .c{background:#eef2ff;}
.tbl-head .c{padding:11px 14px;font-size:11px;font-weight:600;color:#6b7694;text-transform:uppercase;letter-spacing:.5px;white-space:nowrap;display:flex;align-items:center;justify-content:flex-end;}
.tbl-head .c:first-child{justify-content:flex-start;}
.tbl-row .c{padding:12px 14px;display:flex;align-items:center;justify-content:flex-end;white-space:nowrap;}
.tbl-row .c:first-child{justify-content:flex-start;}
.tbl-row.tbl-total .c{padding:13px 14px;font-weight:700;}
.month{font-weight:600;font-size:13.5px;}
.total-label{font-weight:700;font-size:12px;color:#4361ee;text-transform:uppercase;letter-spacing:.5px;}
.fv{font-weight:700;font-size:14px;font-variant-numeric:tabular-nums;}
.fv.up{color:#16a34a;}
.fv.dn{color:#dc2626;}
.pv{color:#4361ee;font-weight:600;font-variant-numeric:tabular-nums;}
.abw-wrap{display:flex;align-items:center;gap:5px;}
.abw-t{font-size:12px;font-weight:500;font-variant-numeric:tabular-nums;}
.abw-t.up{color:#16a34a;}
.abw-t.dn{color:#dc2626;}
.abw-p{font-size:11.5px;font-weight:600;padding:2px 7px;border-radius:4px;min-width:54px;text-align:center;font-variant-numeric:tabular-nums;}
.abw-p.up{color:#16a34a;background:#f0fdf4;}
.abw-p.dn{color:#dc2626;background:#fff1f1;}
.arr{font-size:11px;}
.arr.up{color:#16a34a;}
.arr.dn{color:#dc2626;}
.num{font-variant-numeric:tabular-nums;color:#374151;}
.mc-ftr{padding:13px 28px;background:#f8f9fb;border-top:1px solid #e8ecf1;font-size:12px;color:#6b7694;display:flex;gap:16px;}
.mc-ftr strong{color:#1e2430;font-weight:600;}
</style>

<div class="mc">
  <div class="mc-hdr">
    <span class="mc-title">Production Metrics &#8212; Fiscal Year Overview</span>
    <span class="mc-badge">&#9654; Plan Feed: 575 t/Tag</span>
  </div>
  <div class="tbl-wrap">
    <div class="tbl-head">
      <div class="c">Month</div>
      <div class="c">Feed IST</div>
      <div class="c">Plan</div>
      <div class="c">Abw. (t)</div>
      <div class="c">Abw. (%)</div>
      <div class="c">Oil</div>
      <div class="c">Meal</div>
    </div>
    {{#each data}}
    <div class="tbl-row {{#if (eq fisc_month "99 Gesamt")}}tbl-total{{/if}}">
      <div class="c">
        <span class="{{#if (eq fisc_month "99 Gesamt")}}total-label{{else}}month{{/if}}">{{#if (eq fisc_month "99 Gesamt")}}&#931; Gesamt{{else}}{{fisc_month}}{{/if}}</span>
      </div>
      <div class="c">
        <span class="fv {{#if (gte this.[SUM(feed_tons)] this.[SUM(plan_tons)])}}up{{else}}dn{{/if}}">{{this.[SUM(feed_tons)]}} t</span>
      </div>
      <div class="c">
        <span class="pv">{{this.[SUM(plan_tons)]}} t</span>
      </div>
      <div class="c">
        <div class="abw-wrap">
          <span class="arr {{#if (gte this.[SUM(feed_tons)] this.[SUM(plan_tons)])}}up{{else}}dn{{/if}}">{{#if (gte this.[SUM(feed_tons)] this.[SUM(plan_tons)])}}&#9650;{{else}}&#9660;{{/if}}</span>
          <span class="abw-t {{#if (gte this.[SUM(feed_tons)] this.[SUM(plan_tons)])}}up{{else}}dn{{/if}}">{{#if (gte this.[SUM(feed_tons)] this.[SUM(plan_tons)])}}+{{/if}}{{this.[SUM(abw_tons)]}} t</span>
        </div>
      </div>
      <div class="c">
        <span class="abw-p {{#if (gte this.[SUM(feed_tons)] this.[SUM(plan_tons)])}}up{{else}}dn{{/if}}">{{#if (gte this.[SUM(feed_tons)] this.[SUM(plan_tons)])}}+{{/if}}{{this.[AVG(abw_pct)]}} %</span>
      </div>
      <div class="c"><span class="num">{{this.[SUM(oil_tons)]}} t</span></div>
      <div class="c"><span class="num">{{this.[SUM(meal_tons)]}} t</span></div>
    </div>
    {{/each}}
  </div>
  <div class="mc-ftr">
    Plan: <strong>575 t/Tag</strong>
    &nbsp;&#183;&nbsp; &#9650; Gr&#252;n = IST &#8805; Plan
    &nbsp;&#183;&nbsp; &#9660; Rot = IST &lt; Plan
  </div>
</div>
```

---

## Adapting for a new table

| What to change | Where |
|---|---|
| Plan value (575) | SQL query + template badge + footer |
| Column names | SQL aliases → chart metrics → `this.[SUM(col)]` in template |
| Number of columns | `grid-template-columns` in CSS + header divs + row divs |
| Colors (green/red) | `.up` and `.dn` classes in `<style>` |
| Title / badge text | `.mc-title` and `.mc-badge` spans |
| Font | `font-family` in `.mc` class |
| Totals row label | `'99 Gesamt'` in SQL + `{{#if (eq fisc_month "99 Gesamt")}}` in template |

---

## Handlebars helpers available in Superset

`eq`, `ne`, `lt`, `gt`, `lte`, `gte`, `and`, `or`, `not`, `includes`, `typeof`

Built-in — no custom registration needed. `{{#unless}}` is **not** supported.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| CSS appears as raw text | CSS Styles field not empty | Clear the CSS Styles field |
| `<style>` content visible as text | `HTML_SANITIZATION` not set | Add `HTML_SANITIZATION = False` to superset_config.py + restart |
| Table rows show as raw HTML | `<table>` used | Replace with CSS Grid divs |
| Values empty (`{{feed_tons}} t`) | Wrong column name | Use `{{this.[SUM(feed_tons)]}}` bracket notation |
| Content inside `{{#if}}` shows as raw HTML | `{{#if}}` wrapping `<div>`/`<span>` elements | Move `{{#if}}` inside attributes and text nodes only |
| `{{#unless}}` block renders as raw HTML | `{{#unless}}` not supported | Restructure using `{{#if (ne ...)}}...{{else}}...{{/if}}` |
| Conditional CSS class broken | String column used as metric | Use `{{#if (gte ...)}}` numeric comparison instead |
| HTML after comment shows as text | HTML comment in template | Remove all `<!-- ... -->` comments |
| Div with inline style disappears | `style="..."` on div stripped | Use a CSS class instead |
