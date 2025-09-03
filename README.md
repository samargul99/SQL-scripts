# Portfolio Dashboards & SQL Scripts

This repository documents how I built **Excel and Power BI dashboards** and wrote **SQL queries** for my case studies.  
It contains:  
- 📂 `/sql` → schema + analysis queries  
- 📂 `/powerbi` → dashboard build steps & DAX  
- 📊 Screenshots and project descriptions are in my Notion portfolio.  

---

## SQL Scripts
- `schema.sql` → Minimal schema for SKUs, BOM costs, vendors, and price trends.  
- `cogs_analysis.sql` → Compare SKU versions (v1 vs v2), calculate savings.  
- `vendor_trends.sql` → Vendor Alpha vs Beta pricing trends, savings if switching.  
- `helpers.sql` → Views for latest BOMs, version comparison, and monthly variance.  

---

## Power BI Dashboards
- `cogs_optimization.md` → KPI cards + charts for SKU cost reductions.  
- `vendor_comparison.md` → Vendor Alpha vs Beta comparison and savings %.  
- `price_trends.md` → Line chart of ingredient price trends.  
- `nmpa_submission.md` → Regulatory readiness KPIs, submission tracking, and compliance monitoring.  

---

## Note
All datasets are **synthetic** or dummy recreations for portfolio purposes.  
No proprietary data is included.

## Portfolio Schema for COGS & Vendor Analysis

CREATE TABLE skus (
  sku_id        VARCHAR(20) PRIMARY KEY,
  sku_name      VARCHAR(120),
  category      VARCHAR(80)
);

CREATE TABLE bom_costs (
  sku_id      VARCHAR(20),
  version_no  INT,
  bom_cost    DECIMAL(14,2),
  PRIMARY KEY (sku_id, version_no)
);

CREATE TABLE vendors (
  vendor_id    VARCHAR(20) PRIMARY KEY,
  vendor_name  VARCHAR(120)
);

CREATE TABLE price_trends (
  ingredient_name  VARCHAR(120),
  vendor_name      VARCHAR(120),
  price_date       DATE,
  price_per_uom    DECIMAL(12,2),
  PRIMARY KEY (ingredient_name, vendor_name, price_date)
);

-- Compare SKU version costs and savings

WITH ranked AS (
  SELECT
    sku_id,
    version_no,
    bom_cost,
    ROW_NUMBER() OVER (PARTITION BY sku_id ORDER BY version_no ASC)  AS rn_asc,
    ROW_NUMBER() OVER (PARTITION BY sku_id ORDER BY version_no DESC) AS rn_desc
  FROM bom_costs
),
base AS (
  SELECT
    r1.sku_id,
    r1.bom_cost AS cost_v1,
    r2.bom_cost AS cost_v2
  FROM ranked r1
  JOIN ranked r2
    ON r1.sku_id = r2.sku_id
   AND r1.rn_asc = 1
   AND r2.rn_desc = 1
)
SELECT
  s.sku_id,
  s.sku_name,
  b.cost_v1,
  b.cost_v2,
  (b.cost_v1 - b.cost_v2)                    AS savings_usd,
  ROUND((b.cost_v1 - b.cost_v2)/b.cost_v1*100, 2) AS savings_pct
FROM base b
JOIN skus s ON s.sku_id = b.sku_id
ORDER BY savings_usd DESC;

## Vendor Alpha vs Beta Comparison

WITH latest AS (
  SELECT MAX(price_date) AS max_date FROM price_trends
)
SELECT
  ingredient_name,
  MAX(CASE WHEN vendor_name='Vendor Alpha' THEN price_per_uom END) AS alpha_price,
  MAX(CASE WHEN vendor_name='Vendor Beta'  THEN price_per_uom END) AS beta_price,
  ROUND(
    (MAX(CASE WHEN vendor_name='Vendor Alpha' THEN price_per_uom END) -
     MAX(CASE WHEN vendor_name='Vendor Beta'  THEN price_per_uom END))
    / MAX(CASE WHEN vendor_name='Vendor Alpha' THEN price_per_uom END) * 100, 2
  ) AS savings_if_switch_pct
FROM price_trends p
JOIN latest l ON p.price_date = l.max_date
GROUP BY ingredient_name
ORDER BY savings_if_switch_pct DESC;

## Helpful Views

## Latest BOM per SKU
CREATE OR REPLACE VIEW v_latest_bom AS
SELECT DISTINCT ON (sku_id)
  sku_id, version_no, bom_cost
FROM bom_costs
ORDER BY sku_id, version_no DESC;

## Version Comparison View
CREATE OR REPLACE VIEW v_version_compare AS
WITH first_ver AS (
  SELECT DISTINCT ON (sku_id) sku_id, version_no AS v1, bom_cost AS cost_v1
  FROM bom_costs ORDER BY sku_id, version_no ASC
),
last_ver AS (
  SELECT DISTINCT ON (sku_id) sku_id, version_no AS v2, bom_cost AS cost_v2
  FROM bom_costs ORDER BY sku_id, version_no DESC
)
SELECT
  f.sku_id,
  f.cost_v1,
  l.cost_v2,
  (f.cost_v1 - l.cost_v2) AS savings_usd,
  ROUND((f.cost_v1 - l.cost_v2)/f.cost_v1*100,2) AS savings_pct
FROM first_ver f
JOIN last_ver l ON l.sku_id = f.sku_id;




