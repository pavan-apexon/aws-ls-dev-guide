# 08 — Athena Views and SQL

> **Quick reference:** [View creation pattern](#view-creation-pattern) · [CTE pattern](#cte-pattern) · [Deduplication via pre-aggregation](#deduplication-via-pre-aggregation) · [Date functions](#date-functions) · [Views inventory](#views-inventory) · [Creating views via CLI](#creating-views-via-cli)

---

## Overview

All analytical queries are pre-computed as Athena views. Lambdas and agents query these views instead of writing ad-hoc JOINs each time. This keeps business logic in one place (the SQL files) and makes Lambda code simple.

Views live in `terraform/sql/`. They are created by `deploy.sh` after Terraform provisions the Glue database.

---

## View Creation Pattern

All views use `CREATE OR REPLACE VIEW` so they can be re-run idempotently on every deploy.

```sql
CREATE OR REPLACE VIEW "view_name" AS
WITH
cte_one AS (
    ...
),
cte_two AS (
    ...
)
SELECT ...
FROM cte_one
JOIN cte_two ON ...
```

**Quoting views:** Use double-quotes around the view name if it contains underscores or reserved words. Athena/Presto syntax requires this.

---

## CTE Pattern

Every view uses CTEs instead of nested subqueries. This makes query plans readable and prevents row multiplication bugs.

**The `monthly_sales_view` shows the full pattern:**

```sql
CREATE OR REPLACE VIEW monthly_sales_view AS
WITH

-- Step 1: Pre-aggregate discounts BEFORE joining to orders
-- (avoids fan-out multiplication when one order has multiple discount rows)
discounts_agg AS (
    SELECT
        order_id,
        SUM(discount_amount)                                                    AS total_discount,
        SUM(CASE WHEN discount_type = 'promotion' THEN discount_amount ELSE 0 END) AS promo_discount,
        SUM(CASE WHEN discount_type = 'loyalty'   THEN discount_amount ELSE 0 END) AS loyalty_discount,
        SUM(CASE WHEN discount_type = 'clearance' THEN discount_amount ELSE 0 END) AS clearance_discount
    FROM order_discounts
    GROUP BY order_id          -- ← one result row per order_id
),

-- Step 2: Join to orders after aggregating
orders_with_discounts AS (
    SELECT
        o.product_id,
        c.customer_type,
        YEAR(o.order_date)                      AS year,
        MONTH(o.order_date)                     AS month,
        DATE_TRUNC('month', o.order_date)       AS year_month,
        o.gross_sales,
        o.gross_sales - COALESCE(d.total_discount, 0.0) AS net_sales,
        COALESCE(d.total_discount, 0.0)         AS total_discount,
        COALESCE(d.promo_discount, 0.0)         AS promo_discount,
        COALESCE(d.loyalty_discount, 0.0)       AS loyalty_discount,
        o.channel,
        o.region_code
    FROM orders o
    LEFT JOIN discounts_agg d ON o.order_id = d.order_id
    INNER JOIN customers c   ON o.customer_id = c.customer_id
),

-- Step 3: Aggregate by grouping dimensions
monthly_totals AS (
    SELECT
        product_id,
        customer_type,
        year_month,
        SUM(gross_sales)    AS gross_sales,
        SUM(net_sales)      AS net_sales,
        SUM(total_discount) AS total_discount
    FROM orders_with_discounts
    GROUP BY product_id, customer_type, year_month
),

-- Step 4: Join product dimension for human-readable names
product_info AS (
    SELECT product_id, product_name, category
    FROM products
)

-- Final SELECT
SELECT
    mt.product_id,
    pi.product_name,
    pi.category,
    mt.customer_type,
    mt.year_month,
    mt.gross_sales,
    mt.net_sales,
    mt.total_discount,
    CASE
        WHEN mt.gross_sales = 0 THEN 0.0
        ELSE ROUND(mt.total_discount / mt.gross_sales * 100, 2)
    END                                              AS discount_pct
FROM monthly_totals mt
INNER JOIN product_info pi ON mt.product_id = pi.product_id
```

---

## Deduplication via Pre-aggregation

The most common bug in multi-table Athena queries: **row multiplication when joining to a table with multiple rows per key**.

**Wrong** — joins discounts directly to orders (one order can have many discount rows → row count explodes):
```sql
-- BAD: order joined to N discount rows → N copies of gross_sales
SELECT o.product_id, SUM(o.gross_sales), SUM(d.discount_amount)
FROM orders o
LEFT JOIN order_discounts d ON o.order_id = d.order_id
GROUP BY o.product_id
```

**Correct** — aggregate discounts first, then join:
```sql
-- GOOD: one discount row per order before joining
WITH discounts_agg AS (
    SELECT
        order_id,
        SUM(discount_amount)                                            AS total_discount_amount,
        SUM(CASE WHEN discount_type = 'promotion'  THEN discount_amount ELSE 0 END) AS promotion_amount,
        SUM(CASE WHEN discount_type = 'loyalty'    THEN discount_amount ELSE 0 END) AS loyalty_amount,
        SUM(CASE WHEN discount_type = 'clearance'  THEN discount_amount ELSE 0 END) AS clearance_amount
    FROM order_discounts
    GROUP BY order_id          -- ← one result row per order_id
)
SELECT
    p.product_name,
    p.product_id,
    SUM(o.gross_sales)                                AS total_gross_sales,
    SUM(COALESCE(d.total_discount_amount, 0))          AS total_discounts,
    SUM(COALESCE(d.promotion_amount, 0))               AS promotion_amount
FROM orders o
LEFT JOIN discounts_agg d ON o.order_id = d.order_id  -- ← safe: 1-to-1
LEFT JOIN products p ON o.product_id = p.product_id
GROUP BY p.product_name, p.product_id
```

This is the exact pattern used in `anomaly_detection_view.sql`.

---

## Date Functions

Athena uses Presto/Trino SQL. These are the date functions used across all views:

| Function | Example | Result |
|---|---|---|
| `YEAR(date)` | `YEAR(order_date)` | `2025` |
| `MONTH(date)` | `MONTH(order_date)` | `6` |
| `QUARTER(date)` | `QUARTER(order_date)` | `2` |
| `DATE_TRUNC('month', date)` | `DATE_TRUNC('month', '2025-06-15')` | `2025-06-01` |
| `DATE_TRUNC('quarter', date)` | `DATE_TRUNC('quarter', '2025-06-15')` | `2025-04-01` |
| `SEQUENCE(start, end, step)` | `SEQUENCE(dt1, dt2, INTERVAL '1' MONTH)` | Array of dates |
| `UNNEST(array)` | `CROSS JOIN UNNEST(seq) AS t(col)` | Explode array to rows |
| `CONCAT(...)` | `CONCAT(CAST(year AS VARCHAR), '-Q', CAST(quarter AS VARCHAR))` | `'2025-Q2'` |
| `NULLIF(a, b)` | `NULLIF(gross_revenue, 0)` | `NULL` if 0, prevents /0 |
| `COALESCE(a, b)` | `COALESCE(d.amount, 0.0)` | 0 if NULL (LEFT JOIN miss) |

---

## Views Inventory

| File | View Name | Purpose |
|------|-----------|---------|
| `monthly_sales_view.sql` | `monthly_sales_view` | Monthly sales by product + customer type — base view |
| `quarterly_revenue_view.sql` | `quarterly_revenue_view` | Quarterly totals with discount breakdown |
| `channel_performance_view.sql` | `channel_performance_view` | Sales split by channel (online/in-store/wholesale) |
| `region_performance_view.sql` | `region_performance_view` | Sales split by region |
| `customer_segment_view.sql` | `customer_segment_view` | Sales by customer type (B2B/B2C/Wholesale) |
| `anomaly_detection_view.sql` | `anomaly_detection_view` | Period-over-period deviations — anomaly agent input |
| `trend_analysis_view.sql` | `trend_analysis_view` | Rolling averages + % change — trend agent input |

**Creation order matters.** `trend_analysis_view` depends on `monthly_sales_view`, so base views must be created first. `deploy.sh` includes `sleep 5` waits between groups to allow Athena to register the view before the next dependent view queries it.

```bash
# deploy.sh — creation order in create_athena_views()
execute_query "sql/monthly_sales_view.sql"       # ← base view, created first
sleep 5
execute_query "sql/quarterly_revenue_view.sql"
execute_query "sql/channel_performance_view.sql"
execute_query "sql/region_performance_view.sql"
execute_query "sql/customer_segment_view.sql"
sleep 5
execute_query "sql/anomaly_detection_view.sql"   # ← depends on monthly_sales_view
execute_query "sql/trend_analysis_view.sql"      # ← depends on monthly_sales_view
```

---

## Creating Views via CLI

The `execute_query` function in `deploy.sh` submits each SQL file to Athena:

```bash
execute_query() {
    local FILE=$1
    local NAME=$2
    echo "Creating view: ${NAME}"

    aws athena start-query-execution \
        --query-string "$(cat ${FILE})" \
        --result-configuration OutputLocation="${ATHENA_OUTPUT}" \
        --query-execution-context Database="${DATABASE_NAME}" \
        --region "${AWS_REGION}"
}
```

Both `DATABASE_NAME` and `ATHENA_OUTPUT` are fetched from SSM by `deploy.sh` using `fetch_param`:

```bash
DATABASE_NAME=$(fetch_param "/${PROJECT_NAME}/data/athena-database")
ATHENA_OUTPUT=$(fetch_param  "/${PROJECT_NAME}/athena/output-location")
```

---

## Adding a New View

1. Create `terraform/sql/{view_name}.sql` with `CREATE OR REPLACE VIEW "{view_name}" AS ...`.
2. Add an `execute_query` call at the right point in `create_athena_views()` in `deploy.sh`.
3. If the view depends on another view, add it after that view with an appropriate `sleep` buffer.
4. Run `bash deploy.sh` (or just the Athena section if infrastructure is already deployed).

---

## Querying Views from Lambda

Views behave exactly like tables from the query side:

```python
# From a backend Lambda — view returns pre-computed monthly sales data
query = """
    SELECT product_name, product_id, year_month,
           gross_sales, net_sales, discount_pct
    FROM monthly_sales_view
    WHERE product_id = 'P-1042'
    ORDER BY year_month
"""
rows = execute_athena_query(query)
```

---

[← 07: AgentCore Gateway & Identity](07-agentcore-gateway-identity.md) &nbsp;·&nbsp; [🏠 Home](README.md) &nbsp;·&nbsp; [Next: 09 — Dynamic Config (SSM) →](09-dynamic-config-ssm.md)
