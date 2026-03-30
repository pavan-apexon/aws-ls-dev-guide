# 04 — Backend Lambda APIs

> **Quick reference:** [Lambda structure](#lambda-function-structure) · [SSM cold start](#ssm-at-cold-start) · [Athena pattern](#athena-query-and-poll) · [Response helpers](#response-helpers) · [API Gateway routing](#api-gateway-routing)

---

## Overview

Backend Lambdas are thin API handlers: they read config from SSM at cold start, query Athena or DynamoDB per request, and return a JSON response. They contain no business logic — that lives in the agents and forecast processor.

Each Lambda handles **one route** in API Gateway. This keeps packages small (each function only imports what it needs) and makes IAM scoping easy.

---

## Lambda Function Structure

Every backend Lambda follows the same 5-section pattern:

```
1. Imports
2. Logging setup
3. SSM config fetch (cold start — happens once per container lifetime)
4. execute_athena_query() helper
5. lambda_handler() → response()
```

```python
# backend/backend_get_products.py

import json, os, boto3, time, logging
from botocore.exceptions import ClientError

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

AWS_REGION          = os.environ.get('AWS_REGION', 'us-east-1')
ATHENA_POLL_INTERVAL = 2       # seconds between status polls

athena_client = boto3.client("athena", region_name=AWS_REGION)
ssm_client    = boto3.client("ssm",    region_name=AWS_REGION)

# ── Cold-start config ──────────────────────────────────────────────
# Env var holds the SSM parameter NAME (not value) — Lambda fetches
# the actual value on first invocation. This pattern lets Terraform
# rotate values without redeploying the Lambda.

athena_database_param = os.environ.get('ATHENA_DATABASE')
if not athena_database_param:
    raise ValueError("ATHENA_DATABASE environment variable is required")

try:
    ATHENA_DATABASE = ssm_client.get_parameter(Name=athena_database_param)['Parameter']['Value']
except Exception as e:
    logger.error(f"Could not fetch {athena_database_param} from SSM: {e}")
    raise

athena_output_param = os.environ.get('ATHENA_OUTPUT_LOCATION')
if not athena_output_param:
    raise ValueError("ATHENA_OUTPUT_LOCATION environment variable is required")

try:
    ATHENA_OUTPUT = ssm_client.get_parameter(Name=athena_output_param)['Parameter']['Value']
except Exception as e:
    logger.error(f"Could not fetch {athena_output_param} from SSM: {e}")
    raise
```

---

## SSM at Cold Start

The pattern: **env var = SSM parameter name**, Lambda fetches the value.

| Env var set by Terraform | Value in SSM | Lambda action |
|---|---|---|
| `ATHENA_DATABASE=/{project}/data/athena-database` | `retail_analytics_dev_database` | `ssm.get_parameter(Name=ATHENA_DATABASE)` |
| `ATHENA_OUTPUT_LOCATION=/{project}/athena/output-location` | `s3://bucket/` | same |
| `SSM_DYNAMODB_TABLE_PARAM=/{project}/report/results-table` | `retail-analytics-dev-report-results` | same |

**Why use SSM parameter names as env vars instead of values directly?**

1. Terraform only needs to set the parameter name at deploy time — the value can be updated in SSM later without a Lambda redeploy.
2. Secrets (`SecureString`) are fetched with `WithDecryption=True` — the raw secret never touches Terraform state or env var storage.
3. Centralized config — the same SSM path is read by Lambda, agents, and `env_generate.py`.

---

## Athena Query and Poll

Athena is asynchronous: you start a query, get a `QueryExecutionId`, then poll until it finishes.

```python
def execute_athena_query(query: str) -> list[dict]:
    """Execute an Athena query and return rows as list[dict]."""
    resp = athena_client.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": ATHENA_DATABASE},
        ResultConfiguration={"OutputLocation": ATHENA_OUTPUT},
    )
    qid = resp["QueryExecutionId"]
    logger.info(f"[Athena] QueryExecutionId: {qid}")

    # Poll until terminal state
    while True:
        res   = athena_client.get_query_execution(QueryExecutionId=qid)
        state = res["QueryExecution"]["Status"]["State"]
        if state in ("SUCCEEDED", "FAILED", "CANCELLED"):
            break
        time.sleep(ATHENA_POLL_INTERVAL)

    if state != "SUCCEEDED":
        reason = res["QueryExecution"]["Status"].get("StateChangeReason", "No reason provided")
        raise RuntimeError(f"Athena query failed ({state}): {reason}")

    # Get and transform results
    results = athena_client.get_query_results(QueryExecutionId=qid)
    rows    = results["ResultSet"]["Rows"]
    headers = [col.get("VarCharValue", "") for col in rows[0]["Data"]]
    data    = [
        {headers[i]: col.get("VarCharValue") for i, col in enumerate(r["Data"])}
        for r in rows[1:]
    ]
    logger.info(f"[Athena] Retrieved {len(data)} records")
    return data
```

**Caveats:**
- `get_query_results` returns at most 1,000 rows per call. Add `NextToken` pagination for large tables.
- Athena stores result CSV in the output S3 bucket alongside query metadata. This costs a few cents per TB scanned.
- Both `ATHENA_DATABASE` and `ATHENA_OUTPUT` are fetched once at cold start — not on each invocation.

---

## Response Helpers

Every handler uses a shared `response()` function that adds CORS headers. API Gateway requires these for browser calls.

```python
def response(code: int, body) -> dict:
    """Build an API Gateway proxy response with CORS headers."""
    return {
        "statusCode": code,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin":  "*",
            "Access-Control-Allow-Headers": "*",
            "Access-Control-Allow-Methods": "GET,POST,OPTIONS",
        },
        "body": json.dumps(body, default=str),
    }
```

`default=str` is a safety net for `datetime` and other non-JSON-serializable types. For `Decimal` values from DynamoDB, use a dedicated `DecimalEncoder` (see [05 — Async Lambda Architecture](05-async-lambda-architecture.md)).

---

## lambda_handler Pattern

```python
def lambda_handler(event, context):
    # Handle CORS preflight
    if event.get("httpMethod") == "OPTIONS":
        return response(200, {"message": "CORS preflight"})

    try:
        # Example: GET /get-products
        query  = "SELECT DISTINCT product_id, product_name, category, sku FROM products ORDER BY product_name"
        data   = execute_athena_query(query)
        return response(200, {"products": data})

    except RuntimeError as e:
        logger.error(f"Athena error: {e}")
        return response(500, {"error": str(e)})

    except ClientError as e:
        error_code = e.response['Error']['Code']
        logger.error(f"AWS error ({error_code}): {e}")
        return response(500, {"error": "AWS service error", "code": error_code})

    except Exception as e:
        logger.error(f"Unexpected error: {e}", exc_info=True)
        return response(500, {"error": "Internal server error"})
```

---

## Lambda Terraform Declaration

```hcl
# terraform/backend_lambdas.tf

resource "aws_lambda_function" "get_products" {
  function_name = "${var.project_name}-get-products"
  role          = aws_iam_role.lambda_execution_role.arn
  handler       = "backend_get_products.lambda_handler"
  runtime       = var.lambda_runtime
  timeout       = 30
  memory_size   = 256
  layers        = [var.lambda_layer_arn]       # AWSSDKPandas or similar

  filename         = data.archive_file.backend.output_path
  source_code_hash = data.archive_file.backend.output_base64sha256

  environment {
    variables = {
      # Values are SSM parameter NAMES — not values
      ATHENA_DATABASE         = aws_ssm_parameter.athena_database.name
      ATHENA_OUTPUT_LOCATION  = aws_ssm_parameter.athena_output_location.name
    }
  }

  tags = { Environment = var.env, Project = var.project_name }
}

# Zip the backend/ folder
data "archive_file" "backend" {
  type        = "zip"
  source_dir  = "${path.root}/../backend"
  output_path = "${path.module}/.builds/backend.zip"
}
```

---

## Backend Routes

One Lambda function per route. All routes live in a single API Gateway HTTP API (v2).

| Route | Handler file | Key query |
|---|---|---|
| `GET /get-products` | `backend_get_products.py` | `SELECT product_id, product_name, category FROM products` |
| `GET /get-sales-summary` | `backend_get_sales_summary.py` | Athena `monthly_sales_view` query |
| `GET /get-report-data` | `backend_get_report_data.py` | DynamoDB `REPORT#{type}` lookup |
| `GET /get-customer-types` | `backend_get_customer_types.py` | `SELECT DISTINCT customer_type FROM customers` |
| `GET /get-regions` | `backend_get_regions.py` | `SELECT region_code, region_name FROM regions` |
| `GET /get-all-anomalies` | `backend_get_all_anomalies.py` | DynamoDB scan on agent results table |
| `GET /get-anomalies` | `backend_get_anomalies.py` | DynamoDB query for specific product |
| `GET /get-trend-analysis` | `backend_get_trend_analysis.py` | DynamoDB query for trend agent results |
| `POST /start-sales-report` | `report_lambda/initiator.py` | Creates DynamoDB job + invokes worker async |
| `GET /get-report-status` | `report_lambda/status_checker.py` | DynamoDB `job_id` lookup |

---

## API Gateway Routing

HTTP API (v2) with Lambda proxy integration. Each route maps to exactly one Lambda:

```hcl
# terraform/api_gateway.tf

resource "aws_apigatewayv2_api" "main" {
  name          = "${var.project_name}-api"
  protocol_type = "HTTP"
  description   = "Retail Analytics API"

  cors_configuration {
    allow_origins = ["*"]
    allow_methods = ["GET", "POST", "OPTIONS"]
    allow_headers = ["*"]
  }
}

# Example: get-products route
resource "aws_apigatewayv2_integration" "get_products" {
  api_id                 = aws_apigatewayv2_api.main.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.get_products.invoke_arn
  payload_format_version = "1.0"
}

resource "aws_apigatewayv2_route" "get_products" {
  api_id    = aws_apigatewayv2_api.main.id
  route_key = "GET /get-products"
  target    = "integrations/${aws_apigatewayv2_integration.get_products.id}"
}

resource "aws_lambda_permission" "get_products" {
  statement_id  = "AllowApiGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.get_products.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.main.execution_arn}/*/*"
}
```

**Store the API Gateway URL in SSM after creation:**

```hcl
resource "aws_ssm_parameter" "api_gateway_url" {
  name  = "/${var.project_name}/api/gateway-url"
  type  = "String"
  value = aws_apigatewayv2_api.main.api_endpoint
}
```

`env_generate.py` reads this SSM parameter to construct all `VITE_API_*` entries for the UI `.env` file.

---

## Adding a New Backend Route

1. Create `backend/backend_{name}.py` following the 5-section template above.
2. Add an `aws_lambda_function` resource in `backend_lambdas.tf`.
3. Add an `aws_ssm_parameter` for any new config values the Lambda needs.
4. Add 3 blocks (integration + route + permission) in `api_gateway.tf`.
5. Run `terraform apply`.
6. Add the new URL to `env_generate.py` if the UI needs to call it.

---

[← 03: Data Layer](03-data-layer.md) &nbsp;·&nbsp; [🏠 Home](README.md) &nbsp;·&nbsp; [Next: 05 — Async Lambda Architecture →](05-async-lambda-architecture.md)
