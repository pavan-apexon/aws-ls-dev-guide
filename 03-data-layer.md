# 03 — Data Layer

> **Quick reference:** [Source tables](#source-tables) · [Parquet conversion](#csv-to-parquet-conversion) · [Glue catalog](#glue-catalog-tables) · [Athena queries](#athena-query-patterns) · [Data model](#entity-relationships)

---

## Overview

The data layer is static for a deployment: parquet files are uploaded to S3 once by `terraform apply` and queried by backend Lambdas and agents via Athena. The only mutable layer is the DynamoDB tables written by Lambdas at runtime.

```
terraform/data/
├── csv/            # Source CSV files (not committed — your data)
│   ├── transactions.csv
│   ├── product_master.csv
│   └── ...
└── parquet/        # Converted parquet (committed — Terraform uploads these)
    ├── transactions.parquet
    ├── product_master.parquet
    └── ...
```

---

## Source Tables

Nine domain tables cover the full business data model:

| Table | File | Key Columns | Join Keys |
|-------|------|-------------|-----------|
| `orders` | `orders.parquet` | `order_date`, `product_id`, `quantity`, `gross_sales`, `channel`, `region_code` | `order_id`, `customer_id` |
| `products` | `products.parquet` | `product_name`, `category`, `sku`, `unit_cost` | `product_id` |
| `customers` | `customers.parquet` | `customer_name`, `customer_type` (B2B/B2C/Wholesale) | `customer_id` |
| `order_discounts` | `order_discounts.parquet` | `discount_type` (promotion/loyalty/clearance), `discount_amount` | `order_id` |
| `contracts` | `contracts.parquet` | `contract_type`, `discount_rate`, `effective_date`, `expiration_date` | `product_id`, `customer_type` |
| `suppliers` | `suppliers.parquet` | `supplier_name` | `supplier_id` |
| `distributors` | `distributors.parquet` | `distributor_name` | `distributor_id` |
| `regions` | `regions.parquet` | `region_name`, `country_code` | `region_code` |
| `regional_tax_rates` | `regional_tax_rates.parquet` | `tax_rate`, `effective_date` | `region_code`, `product_id` |

---

## CSV-to-Parquet Conversion

Run the conversion script once to produce the parquet files that Terraform will upload.

```bash
cd data
python3 csv_to_parquet.py
```

The script reads every `.csv` from `csv/` and writes a `.parquet` to `parquet/` using `pandas` + `pyarrow`:

```python
# data/csv_to_parquet.py
import pandas as pd
from pathlib import Path

CSV_DIR  = Path("csv")
OUT_DIR  = Path("parquet")
OUT_DIR.mkdir(exist_ok=True)

for csv_file in CSV_DIR.glob("*.csv"):
    df = pd.read_csv(csv_file)
    out = OUT_DIR / csv_file.with_suffix(".parquet").name
    df.to_parquet(out, index=False)
    print(f"Converted: {csv_file.name} → {out.name}  ({len(df)} rows)")
```

Date columns must be native `datetime64` before conversion — not strings — so Athena can use `YEAR()`, `MONTH()`, `DATE_TRUNC()` without casting:

```python
# Example: parse date columns before writing parquet
DATE_COLS = {
    "orders":    ["order_date"],
    "contracts": ["effective_date", "expiration_date"],
    "regional_tax_rates": ["effective_date"],
}

for csv_file in CSV_DIR.glob("*.csv"):
    df   = pd.read_csv(csv_file)
    name = csv_file.stem
    for col in DATE_COLS.get(name, []):
        if col in df.columns:
            df[col] = pd.to_datetime(df[col])
    df.to_parquet(OUT_DIR / f"{name}.parquet", index=False)
```

---

## S3 Layout

Each table is in its own S3 prefix so future partition pruning is easy:

```
s3://{project_name}-{account_id}-data/
└── data/
    ├── orders/orders.parquet
    ├── products/products.parquet
    ├── customers/customers.parquet
    ├── order_discounts/order_discounts.parquet
    ├── contracts/contracts.parquet
    ├── suppliers/suppliers.parquet
    ├── distributors/distributors.parquet
    ├── regions/regions.parquet
    └── regional_tax_rates/regional_tax_rates.parquet
```

Terraform creates each `aws_s3_object` pointing at the prefix-per-table location:

```hcl
resource "aws_s3_object" "orders_data" {
  bucket = aws_s3_bucket.data.id
  key    = "${var.data_prefix}/orders/orders.parquet"
  source = "${path.module}/../data/parquet/orders.parquet"
  etag   = filemd5("${path.module}/../data/parquet/orders.parquet")

  depends_on = [aws_s3_bucket.data, aws_s3_bucket_policy.data_policy]
  tags = { DataType = "Orders" }
}
```

`etag = filemd5(...)` ensures Terraform only re-uploads the file when its content changes.

---

## Glue Catalog Tables

Every Athena query needs a Glue catalog entry that describes the schema and points to the S3 prefix. Use Parquet input/output formats with `ParquetHiveSerDe`.

**Pattern for all tables:**
```hcl
resource "aws_glue_catalog_table" "orders" {
  name          = "orders"
  database_name = aws_glue_catalog_database.main.name
  description   = "Sales order records"
  table_type    = "EXTERNAL_TABLE"

  storage_descriptor {
    location      = "s3://${aws_s3_bucket.data.id}/${var.data_prefix}/orders/"
    input_format  = "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat"
    output_format = "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat"

    ser_de_info {
      serialization_library = "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe"
    }

    columns { name = "order_id",    type = "string" }
    columns { name = "order_date",  type = "date"   }
    columns { name = "customer_id", type = "string" }
    columns { name = "product_id",  type = "string" }
    columns { name = "quantity",    type = "bigint" }
    columns { name = "unit_price",  type = "double" }
    columns { name = "gross_sales", type = "double" }
    columns { name = "channel",     type = "string" }
    columns { name = "region_code", type = "string" }
  }
}
```

**Type mapping — CSV/pandas to Glue:**

| pandas dtype | Glue/HiveQL type |
|---|---|
| `object` (string) | `string` |
| `int64` | `bigint` |
| `float64` | `double` |
| `datetime64[ns]` | `date` or `timestamp` |
| `bool` | `boolean` |

---

## DynamoDB Tables — Runtime Storage

Three DynamoDB tables are created for runtime state (not source data):

| Table | Purpose | Key Pattern | TTL |
|-------|---------|-------------|-----|
| `{project}-report-results` | Generated report output | `PK=REPORT#{report_type}`, `SK=DATE#{period}` | No TTL — permanent |
| `{project}-async-jobs` | Async Lambda job tracking | `job_id` (hash only) | 24 h TTL via `expires_at` attribute |
| `{project}-agent-results` | Agent analysis output | `PK`, `SK` per product+period | No TTL |

All three use `PAY_PER_REQUEST`. See [05 — Async Lambda Architecture](05-async-lambda-architecture.md) for how the async-jobs table works.

---

## Athena Query Patterns

All Athena access goes through this `execute_athena_query()` helper. The SSM `ATHENA_DATABASE` and `ATHENA_OUTPUT` values are fetched once at Lambda cold start.

```python
# At cold start — fetch SSM values ONCE, not per-request
athena_database_param = os.environ.get('ATHENA_DATABASE')
response = ssm_client.get_parameter(Name=athena_database_param)
ATHENA_DATABASE = response['Parameter']['Value']

athena_output_param = os.environ.get('ATHENA_OUTPUT_LOCATION')
response = ssm_client.get_parameter(Name=athena_output_param)
ATHENA_OUTPUT = response['Parameter']['Value']
```

```python
def execute_athena_query(query: str) -> list[dict]:
    """Start query, poll until done, return rows as list of dicts."""
    resp = athena_client.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": ATHENA_DATABASE},
        ResultConfiguration={"OutputLocation": ATHENA_OUTPUT},
    )
    qid = resp["QueryExecutionId"]

    # Poll for completion
    while True:
        res   = athena_client.get_query_execution(QueryExecutionId=qid)
        state = res["QueryExecution"]["Status"]["State"]
        if state in ("SUCCEEDED", "FAILED", "CANCELLED"):
            break
        time.sleep(2)

    if state != "SUCCEEDED":
        reason = res["QueryExecution"]["Status"].get("StateChangeReason", "")
        raise RuntimeError(f"Athena query failed ({state}): {reason}")

    # Convert resultset to list[dict]
    results = athena_client.get_query_results(QueryExecutionId=qid)
    rows    = results["ResultSet"]["Rows"]
    headers = [col.get("VarCharValue", "") for col in rows[0]["Data"]]
    return [
        {headers[i]: col.get("VarCharValue") for i, col in enumerate(r["Data"])}
        for r in rows[1:]
    ]
```

Athena's `GetQueryResults` is synchronous (returns up to 1,000 rows). For large result sets add pagination via `NextToken`.

---

## Entity Relationships

```
orders ──── order_discounts     (order_id)
orders ──── customers           (customer_id)
orders ──── products            (product_id)
orders ──── regions             (region_code)
orders ──── distributors        (distributor_id)
contracts ── products            (product_id)
contracts ── customers           (customer_type)
products ─── suppliers           (supplier_id)
regions ──── regional_tax_rates  (region_code + product_id)
```

---

## Adding a New Data Table

1. Add the CSV to `terraform/data/csv/`.
2. Add a date-column entry to `csv_to_parquet.py` if needed.
3. Run `python3 csv_to_parquet.py` — verify the parquet is in `terraform/data/parquet/`.
4. Add an `aws_s3_object` in `data_infrastructure.tf` with `depends_on = [bucket, policy]`.
5. Add an `aws_glue_catalog_table` with the correct column types.
6. Run `terraform apply` — the new file is uploaded and the catalog is updated in one step.

---

[← 02: Terraform Infrastructure](02-terraform-infrastructure.md) &nbsp;·&nbsp; [🏠 Home](README.md) &nbsp;·&nbsp; [Next: 04 — Backend Lambda APIs →](04-backend-lambda-apis.md)
