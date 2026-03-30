# 05 — Async Lambda Architecture

> **Quick reference:** [Why async?](#why-async) · [Three Lambdas](#the-three-lambda-roles) · [Job ID format](#job-id-format) · [DynamoDB job table](#dynamodb-job-table) · [Status flow](#status-flow) · [Client polling](#client-polling-contract) · [DecimalEncoder](#decimalencoder)

---

## Why Async?

**API Gateway has a hard 29-second integration timeout.** Any Lambda that runs longer than 29 seconds will be killed by API Gateway before it can respond, regardless of the Lambda's own timeout setting.

This affects any operation that does meaningful work: generating a multi-table sales report, processing a bulk data export, running a complex aggregation across millions of rows. These operations routinely take 60–120 seconds.

The async trio pattern solves this by decoupling receipt from execution:

```
Synchronous (broken for long-running tasks):
  Client ──POST──► Initiator ──►[29s timeout] ── Worker dies

Async (correct):
  Client ──POST──► Initiator ──202 + job_id──► Client
                       │
                   (invokes Worker async, returns immediately)
                       │
                   Worker ──► DynamoDB (QUEUED → PROCESSING → COMPLETED)
                       │
  Client ──GET──► StatusChecker ──► DynamoDB ──► result when ready
```

The **Initiator** receives the request and returns a `job_id` in under 1 second. The **Worker** does the heavy computation in the background. The **StatusChecker** lets the client poll until the job is done.

**This pattern applies to ANY long-running Lambda**, not just a specific operation type. If it can take longer than 29 seconds, it needs this pattern.

---

## The Three Lambda Roles

| Lambda | Handler file | Trigger | Returns | Timeout |
|--------|-------------|---------|---------|---------|
| **Initiator** | `report_lambda/initiator.py` | API Gateway POST | `202 { job_id, status: QUEUED }` | 30 s |
| **Worker** | `report_lambda/worker.py` | Event invocation (async) | Nothing — writes to DynamoDB | 900 s |
| **StatusChecker** | `report_lambda/status_checker.py` | API Gateway GET | `200 { status, results? }` | 30 s |

---

## Initiator — Receive and Queue

The Initiator validates the request, creates a DynamoDB job record, then immediately invokes the Worker with `InvocationType='Event'` (fire-and-forget).

```python
# report_lambda/initiator.py (abbreviated)

import hashlib, json, boto3, os
from datetime import datetime, timezone, timedelta

lambda_client = boto3.client('lambda')
dynamodb      = boto3.resource('dynamodb')

WORKER_LAMBDA_ARN          = os.environ['WORKER_LAMBDA_ARN']
SSM_ASYNC_JOBS_TABLE_PARAM = os.environ['SSM_ASYNC_JOBS_TABLE_PARAM']

# Fetch table name once at cold start
ASYNC_JOBS_TABLE_NAME = boto3.client('ssm').get_parameter(
    Name=SSM_ASYNC_JOBS_TABLE_PARAM)['Parameter']['Value']


def lambda_handler(event, context):
    report_type = event.get('queryStringParameters', {}).get('report_type')
    # ... validate report_type ...

    # Build a readable, sortable job ID
    report_hash = hashlib.md5(report_type.encode()).hexdigest()[:8]
    timestamp   = datetime.now(tz=timezone.utc).strftime('%Y%m%d%H%M%S%f')[:-3]
    job_id      = f"REPORT#Generation#{report_hash}#{timestamp}"

    # Write QUEUED record with 24h TTL
    ttl = int((datetime.now(tz=timezone.utc) + timedelta(hours=24)).timestamp())
    dynamodb.Table(ASYNC_JOBS_TABLE_NAME).put_item(Item={
        'job_id':      job_id,
        'report_type': report_type,
        'status':      'QUEUED',
        'created_at':  datetime.now(tz=timezone.utc).isoformat(),
        'updated_at':  datetime.now(tz=timezone.utc).isoformat(),
        'ttl':         ttl,
    })

    # Invoke Worker asynchronously — returns immediately
    lambda_client.invoke(
        FunctionName=WORKER_LAMBDA_ARN,
        InvocationType='Event',           # ← async!
        Payload=json.dumps({
            'job_id':      job_id,
            'report_type': report_type,
        }),
    )

    # Return 202 Accepted — job is running in the background
    return {
        "statusCode": 202,
        "body": json.dumps({
            "job_id":          job_id,
            "status":          "QUEUED",
            "message":         "Report job queued. Use job_id to check status.",
            "status_endpoint": f"/get-report-status?job_id={job_id}",
        }),
    }
```

---

## Job ID Format

```
REPORT#Generation#{report_hash}#{timestamp}
                   ^^^^^^^^^^^^
                   8-char MD5 of report_type

Example: REPORT#Generation#a3f1bc90#20250601143025123
```

The `#` delimiter never appears in report type names, so the ID is unambiguously parseable. The timestamp fragment provides natural sort order across concurrent jobs of the same type.

---

## Worker — Process and Store

The Worker runs the actual computation. In this example it generates a complex sales report by querying multiple Athena views and joining results — a task that takes 60–120 seconds and is impossible to complete within the 29-second API Gateway limit.

```python
# report_lambda/worker.py (abbreviated)

import json, boto3, os
from datetime import datetime, timezone

dynamodb = boto3.resource('dynamodb')

SSM_ASYNC_JOBS_TABLE_PARAM = os.environ['SSM_ASYNC_JOBS_TABLE_PARAM']
ASYNC_JOBS_TABLE_NAME = boto3.client('ssm').get_parameter(
    Name=SSM_ASYNC_JOBS_TABLE_PARAM)['Parameter']['Value']


def update_job_status(table, job_id: str, status: str, error_message: str = None):
    update_expr = "SET #s = :s, updated_at = :u"
    expr_values = {
        ':s': status,
        ':u': datetime.now(tz=timezone.utc).isoformat(),
    }
    if error_message:
        update_expr += ", error_message = :e"
        expr_values[':e'] = error_message

    table.update_item(
        Key={'job_id': job_id},
        UpdateExpression=update_expr,
        ExpressionAttributeNames={'#s': 'status'},  # 'status' is a reserved word
        ExpressionAttributeValues=expr_values,
    )


def lambda_handler(event, context):
    job_id      = event['job_id']
    report_type = event['report_type']
    table       = dynamodb.Table(ASYNC_JOBS_TABLE_NAME)

    try:
        # Update status: QUEUED → PROCESSING
        update_job_status(table, job_id, 'PROCESSING')

        # Do the heavy computation (queries Athena views, joins, aggregates)
        result = generate_report(report_type)

        # Update status: PROCESSING → COMPLETED, write result
        table.update_item(
            Key={'job_id': job_id},
            UpdateExpression="SET #s = :s, updated_at = :u, result = :r",
            ExpressionAttributeNames={'#s': 'status'},
            ExpressionAttributeValues={
                ':s': 'COMPLETED',
                ':u': datetime.now(tz=timezone.utc).isoformat(),
                ':r': result,
            },
        )

    except Exception as e:
        update_job_status(table, job_id, 'FAILED', str(e))
        raise
```

**Key point:** `status` is a DynamoDB reserved word. Always use an `ExpressionAttributeNames` alias (`#s = :s`) when updating it.

---

## StatusChecker — Poll and Return

```python
# report_lambda/status_checker.py (abbreviated)

import json, boto3, os
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')

SSM_ASYNC_JOBS_TABLE_PARAM = os.environ['SSM_ASYNC_JOBS_TABLE_PARAM']
SSM_DYNAMODB_TABLE_PARAM   = os.environ['SSM_DYNAMODB_TABLE_PARAM']

ASYNC_JOBS_TABLE_NAME = boto3.client('ssm').get_parameter(Name=SSM_ASYNC_JOBS_TABLE_PARAM)['Parameter']['Value']
REPORT_RESULTS_TABLE  = boto3.client('ssm').get_parameter(Name=SSM_DYNAMODB_TABLE_PARAM)['Parameter']['Value']


class DecimalEncoder(json.JSONEncoder):
    """DynamoDB stores numbers as Decimal — convert to float for JSON."""
    def default(self, obj):
        if isinstance(obj, Decimal):
            return float(obj)
        return super().default(obj)


def lambda_handler(event, context):
    job_id = (event.get('queryStringParameters') or {}).get('job_id')
    if not job_id:
        return {"statusCode": 400, "body": json.dumps({"error": "job_id required"})}

    jobs_table = dynamodb.Table(ASYNC_JOBS_TABLE_NAME)
    response   = jobs_table.get_item(Key={'job_id': job_id})
    job        = response.get('Item')

    if not job:
        return {"statusCode": 404, "body": json.dumps({"error": "Job not found"})}

    status = job.get('status', 'UNKNOWN')

    if status == 'COMPLETED':
        # Fetch the actual results from the report results table
        results = fetch_report_results(
            report_type=job.get('report_type'),
        )
        return {
            "statusCode": 200,
            "body": json.dumps({"job_id": job_id, "status": status, "results": results}, cls=DecimalEncoder),
        }

    if status == 'FAILED':
        return {
            "statusCode": 200,
            "body": json.dumps({
                "job_id": job_id,
                "status": status,
                "error":  job.get('error_message'),
            }),
        }

    # Still QUEUED or PROCESSING
    return {
        "statusCode": 200,
        "body": json.dumps({"job_id": job_id, "status": status}),
    }
```

---

## Status Flow

```
QUEUED       → set by Initiator immediately after put_item
PROCESSING   → set by Worker at start of forecast_all_models()
COMPLETED    → set by Worker after all models finish + results written
FAILED       → set by Worker in except block with error_message
```

---

## DynamoDB Job Table

```hcl
# terraform/data_infrastructure.tf

resource "aws_dynamodb_table" "async_jobs" {
  name         = "${var.project_name}-async-jobs"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "job_id"

  attribute {
    name = "job_id"
    type = "S"
  }

  # Auto-expire records 24 h after creation
  ttl {
    attribute_name = "ttl"      # Lambda sets this to epoch_now + 86400
    enabled        = true
  }

  tags = { Name = "Async Jobs", Environment = var.env, Project = var.project_name }
}
```

**TTL attribute:** Set `ttl = int((datetime.now(tz=timezone.utc) + timedelta(hours=24)).timestamp())` in the Initiator. DynamoDB will delete the record automatically after 24 hours (usually within a few minutes of expiry).

---

## Client Polling Contract

```
POST /start-sales-report?report_type=regional-quarterly
   → 202 { job_id: "REPORT#Generation#a3f1bc90#...", status: "QUEUED" }

GET  /get-report-status?job_id=REPORT#Generation#a3f1bc90#...
   → 200 { status: "PROCESSING" }         # still running — poll again

GET  /get-report-status?job_id=REPORT#Generation#a3f1bc90#...
   → 200 { status: "COMPLETED", results: {...} }  # done
```

Recommended polling interval: 3–5 seconds. The UI implements this as a `setInterval` that stops when `status === 'COMPLETED'` or `'FAILED'`.

---

## DecimalEncoder

DynamoDB stores all numeric types as `Decimal`. The standard `json.dumps()` cannot serialize `Decimal`. Always use `DecimalEncoder` when returning DynamoDB results in a Lambda response:

```python
class DecimalEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Decimal):
            return float(obj)
        return super().default(obj)

# Usage
json.dumps({"value": Decimal("123.45")}, cls=DecimalEncoder)
# → '{"value": 123.45}'
```

---

## Extending the Async Pattern

To add a new async job type (e.g., bulk CSV export, nightly data sync):

1. Create `{name}_initiator.py` — validate input, write DynamoDB job, invoke worker.
2. Create `{name}_worker.py` — do the heavy compute, update job status.
3. Reuse `report_lambda/status_checker.py` unchanged (it reads from the same jobs table by `job_id`).
4. Add `WORKER_LAMBDA_ARN` env var on the Initiator pointing to the new Worker ARN.
5. Configure the Worker Lambda with a 900 s timeout and enough memory for the compute task.

Any operation that **might** exceed 29 seconds should use this pattern. The cost of the async pattern (DynamoDB writes, extra Lambda invocations) is negligible compared to the reliability it provides.

---

[← 04: Backend Lambda APIs](04-backend-lambda-apis.md) &nbsp;·&nbsp; [🏠 Home](README.md) &nbsp;·&nbsp; [Next: 06 — Bedrock Agents →](06-bedrock-agents-strands.md)
