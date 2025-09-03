# SQL-scripts
schema + analysis queries
## Minimal, portfolio-friendly schema for COGS & vendor analysis

## SKUs & versions
CREATE TABLE skus (
  sku_id        VARCHAR(20) PRIMARY KEY,
  sku_name      VARCHAR(120),
  category      VARCHAR(80)
);
CREATE TABLE sku_versions (
  sku_id        VARCHAR(20) REFERENCES skus(sku_id),
  version_no    INT,
  effective_date DATE,
  PRIMARY KEY (sku_id, version_no)
);
## Ingredients
CREATE TABLE ingredients (
  ingredient_id   VARCHAR(20) PRIMARY KEY,
  ingredient_name VARCHAR(120)
);
## Many-to-many: SKU uses Ingredients
CREATE TABLE sku_ingredients (
  sku_id          VARCHAR(20),
  ingredient_id   VARCHAR(20),
  concentration_pct DECIMAL(6,2), 
  ## optional, for contribution visuals
  PRIMARY KEY (sku_id, ingredient_id)
);

## Vendors & prices
CREATE TABLE vendors (
  vendor_id    VARCHAR(20) PRIMARY KEY,
  vendor_name  VARCHAR(120)
);

CREATE TABLE vendor_prices (
  vendor_id      VARCHAR(20) REFERENCES vendors(vendor_id),
  ingredient_id  VARCHAR(20) REFERENCES ingredients(ingredient_id),
  price_date     DATE,
  price_per_uom  DECIMAL(12,2),
  PRIMARY KEY (vendor_id, ingredient_id, price_date)
);

## BOM costs per SKU version (post-reformulation tracking)
CREATE TABLE bom_costs (
  sku_id      VARCHAR(20),
  version_no  INT,
  bom_cost    DECIMAL(14,2),
  PRIMARY KEY (sku_id, version_no)
);
## Optional: price trends for direct charting in Power BI
CREATE TABLE price_trends (
  ingredient_name  VARCHAR(120),
  vendor_name      VARCHAR(120),
  price_date       DATE,
  price_per_uom    DECIMAL(12,2)
);

================
COGS: Version compare & savings
=================

## Latest two versions per SKU (v1, v2)
WITH ranked AS (
  SELECT
    b.sku_id,
    b.version_no,
    b.bom_cost,
    ROW_NUMBER() OVER (PARTITION BY b.sku_id ORDER BY b.version_no ASC)  AS rn_asc,
    ROW_NUMBER() OVER (PARTITION BY b.sku_id ORDER BY b.version_no DESC) AS rn_desc
  FROM bom_costs b
),
base AS (
  SELECT
    r1.sku_id,
    r1.bom_cost AS cost_v1,
    r2.bom_cost AS cost_v2
  FROM ranked r1
  JOIN ranked r2
    ON r1.sku_id = r2.sku_id
   AND r1.rn_asc = 1       -- first/older
   AND r2.rn_desc = 1      -- latest/newer
)
SELECT
  s.sku_id,
  s.sku_name,
  b.cost_v1,
  b.cost_v2,
  (b.cost_v1 - b.cost_v2)                    AS savings_usd,
  CASE WHEN b.cost_v1 > 0
       THEN ROUND((b.cost_v1 - b.cost_v2)/b.cost_v1*100, 2)
       ELSE NULL END                         AS savings_pct
FROM base b
JOIN skus s ON s.sku_id = b.sku_id
ORDER BY savings_usd DESC;

-- Ingredient contribution (top drivers) for a given SKU
-- (Assumes you’ve materialized ingredient costs per SKU-version)
-- Example view join point: sku_ingredients × vendor_prices (chosen vendor) etc.

============================
Vendor comparison & switching
============================ */

-- Average price per vendor for an ingredient over a period
SELECT
  ingredient_name,
  vendor_name,
  DATE_TRUNC('month', price_date) AS month,
  AVG(price_per_uom)              AS avg_price
FROM price_trends
GROUP BY 1,2,3
ORDER BY 1,2,3;

-- Alpha vs Beta head-to-head per ingredient (latest month)
WITH latest AS (
  SELECT MAX(price_date) AS max_date FROM price_trends
)
SELECT
  p.ingredient_name,
  MAX(CASE WHEN p.vendor_name='Vendor Alpha' THEN p.price_per_uom END) AS alpha_price,
  MAX(CASE WHEN p.vendor_name='Vendor Beta'  THEN p.price_per_uom END) AS beta_price,
  CASE
    WHEN MAX(CASE WHEN p.vendor_name='Vendor Alpha' THEN p.price_per_uom END) > 0
    THEN ROUND(
      (MAX(CASE WHEN p.vendor_name='Vendor Alpha' THEN p.price_per_uom END) -
       MAX(CASE WHEN p.vendor_name='Vendor Beta'  THEN p.price_per_uom END))
      / MAX(CASE WHEN p.vendor_name='Vendor Alpha' THEN p.price_per_uom END) * 100, 2)
    ELSE NULL
  END AS savings_if_switch_pct
FROM price_trends p
JOIN latest l ON p.price_date = l.max_date
GROUP BY p.ingredient_name
ORDER BY savings_if_switch_pct DESC NULLS LAST;

-- Timeseries for line chart: just select ingredient_name, vendor_name, price_date, price_per_uom
SELECT ingredient_name, vendor_name, price_date, price_per_uom
FROM price_trends
ORDER BY ingredient_name, vendor_name, price_date;

-- Helpful views to simplify BI model

-- 1) Latest BOM per SKU
CREATE OR REPLACE VIEW v_latest_bom AS
SELECT DISTINCT ON (b.sku_id)
  b.sku_id, b.version_no, b.bom_cost
FROM bom_costs b
ORDER BY b.sku_id, b.version_no DESC;

-- 2) Version compare (v1 vs latest)
CREATE OR REPLACE VIEW v_version_compare AS
WITH first_ver AS (
  SELECT DISTINCT ON (sku_id)
    sku_id, version_no AS v1, bom_cost AS cost_v1
  FROM bom_costs
  ORDER BY sku_id, version_no ASC
),
last_ver AS (
  SELECT DISTINCT ON (sku_id)
    sku_id, version_no AS v2, bom_cost AS cost_v2
  FROM bom_costs
  ORDER BY sku_id, version_no DESC
)
SELECT
  s.sku_id, s.sku_name,
  f.v1, l.v2,
  f.cost_v1, l.cost_v2,
  (f.cost_v1 - l.cost_v2)                          AS savings_usd,
  CASE WHEN f.cost_v1 > 0
       THEN ROUND((f.cost_v1 - l.cost_v2)/f.cost_v1*100, 2)
       ELSE NULL END                               AS savings_pct
FROM skus s
JOIN first_ver f ON f.sku_id = s.sku_id
JOIN last_ver  l ON l.sku_id = s.sku_id;

## Monthly variance at portfolio level (for trend)
## (Assuming a spend table with forecast/actual; otherwise map from your dataset)
## SELECT month, SUM(actual - forecast) AS variance_amt FROM spend GROUP BY month;

