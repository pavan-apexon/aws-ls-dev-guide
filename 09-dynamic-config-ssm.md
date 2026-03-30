# 09 — Dynamic Config with SSM Parameter Store

> **Quick reference:** [Naming convention](#naming-convention) · [Terraform writes](#terraform-writes) · [Lambda reads](#lambda-reads) · [Agent reads with lru_cache](#agent-reads-with-lru_cache) · [env_generate.py](#env_generatepy--ui-env-file) · [SecureString vs String](#securestring-vs-string)

---

## Overview

SSM Parameter Store is the **config bus** for the entire system. Terraform writes all infrastructure values to SSM immediately after creating them. Lambdas, agents, and the UI env generator all read from SSM at runtime — no hard-coded values flow through environment variables or Terraform outputs.

```
                 Terraform apply
                      ↓ writes
              SSM Parameter Store
             ╱         |         ╲
      Lambda        Agent        env_generate.py
   (cold start)  (cold start)        ↓ writes
                                   UI .env file
```

---

## Naming Convention

All parameters follow the pattern: `/{project_name}/{service}/{param-name}`

```
/{project_name}/
├── data/
│   ├── bucket-name              → S3 data bucket name
│   └── athena-database          → Glue database name
├── athena/
│   └── output-location          → s3://bucket/ for Athena results
├── api/
│   └── gateway-url              → API Gateway base URL
├── report/
│   ├── results-table            → DynamoDB report results table
│   └── jobs-table               → DynamoDB async jobs table
├── agentcore/
│   ├── trend-agent-arn          → agent ARN (written by deploy.sh, not Terraform)
│   ├── anomaly-agent-arn        → same
│   ├── trend-workload-name      → workload identity name for trend agent
│   └── anomaly-workload-name    → workload identity name for anomaly agent
├── gateway/
│   ├── url                      → MCP gateway URL
│   ├── id                       → MCP gateway ID
│   └── role-arn                 → IAM role for gateway execution
├── m2m/
│   ├── client_id                → Cognito M2M app client ID
│   ├── discovery_url            → Cognito OIDC discovery URL
│   ├── oauth_provider_name      → AgentCore OAuth provider name
│   ├── scope                    → OAuth scopes (comma-separated)
│   ├── secret_name              → reference to Secrets Manager key for client secret
│   └── user_pool_id             → Cognito user pool ID
├── ui/
│   ├── client_id                → Cognito web app client ID
│   ├── secret_name              → reference to Secrets Manager key for web secret
│   ├── cloudfront-url           → CloudFront distribution URL
│   ├── s3-bucket-name           → UI static assets bucket name
│   └── cloudfront-distribution-id → for cache invalidation
└── bedrock-agentcore/
    └── role-name                → IAM role name for AgentCore execution
```

---

## Terraform Writes

Write every runtime-relevant value immediately after resource creation. Do not skip anything — if a consumer might need it, write it.

```hcl
# terraform/report_ssm_parameters.tf

# Values computed from resource attributes — written as String type
resource "aws_ssm_parameter" "data_bucket" {
  name        = "/${var.project_name}/data/bucket-name"
  type        = "String"
  value       = aws_s3_bucket.data.bucket
  description = "S3 bucket containing source data"
  tags        = { Environment = var.env, Project = var.project_name }
}

resource "aws_ssm_parameter" "athena_database" {
  name        = "/${var.project_name}/data/athena-database"
  type        = "String"
  value       = local.glue_database_name
  description = "Athena/Glue database name"
  tags        = { Environment = var.env, Project = var.project_name }
}

resource "aws_ssm_parameter" "api_gateway_url" {
  name        = "/${var.project_name}/api/gateway-url"
  type        = "String"
  value       = aws_apigatewayv2_api.main.api_endpoint
  description = "Base URL for all API calls"
  tags        = { Environment = var.env, Project = var.project_name }
}

# Placeholder — overwritten by deploy.sh after agentcore deploy
resource "aws_ssm_parameter" "trend_agent_arn" {
  name        = "/${var.project_name}/agentcore/trend-agent-arn"
  type        = "String"
  value       = "PLACEHOLDER"
  description = "ARN of deployed Trend Analysis AgentCore agent — set by deploy.sh"
  tags        = { Environment = var.env, Project = var.project_name }
}
```

**Placeholder pattern:** For values not known until after an external step (e.g., agent deployment), Terraform creates the parameter with `"PLACEHOLDER"`. The deploy script overwrites it after the external step completes.

---

## Lambda Reads

Lambda reads SSM at cold start, not per-request. Two patterns in use:

### Pattern A — Env var holds parameter name, Lambda fetches value

```python
# At module level — runs once on cold start
SSM_PARAM_NAME = os.environ.get('ATHENA_DATABASE')   # ← parameter NAME from env var
if not SSM_PARAM_NAME:
    raise ValueError("ATHENA_DATABASE env var is required")

ssm_client = boto3.client('ssm')
ATHENA_DATABASE = ssm_client.get_parameter(Name=SSM_PARAM_NAME)['Parameter']['Value']
```

**Why store the parameter name in an env var?** The Lambda only needs to know _which_ SSM key to read, not the value. Rotating or changing values in SSM requires no Lambda redeploy.
### Pattern B — Helper function for optional params

```python
def get_table_name_from_ssm(param_name: str, label: str) -> str:
    try:
        return ssm_client.get_parameter(Name=param_name)['Parameter']['Value']
    except ClientError as e:
        logger.error(f"Failed to get {label} table name from SSM ({param_name}): {e}")
        raise

ASYNC_JOBS_TABLE = get_table_name_from_ssm(os.environ['SSM_ASYNC_JOBS_TABLE_PARAM'],  "async jobs")
REPORT_RESULTS   = get_table_name_from_ssm(os.environ['SSM_DYNAMODB_TABLE_PARAM'],     "report results")
```

---

## Agent Reads with `lru_cache`

Agents can be re-invoked within the same container. Cache SSM reads so repeated invocations don't hit SSM every time.

```python
from functools import lru_cache
import boto3

@lru_cache(maxsize=32)
def get_parameter_value(param_name: str) -> str:
    """Fetch and cache an SSM parameter for the lifetime of this container."""
    ssm = boto3.client('ssm', region_name=AWS_REGION)
    return ssm.get_parameter(Name=param_name, WithDecryption=True)['Parameter']['Value']
```

The `@lru_cache(maxsize=32)` decorator memoizes by `param_name`. The first call fetches from SSM; all subsequent calls with the same name return the cached value in microseconds.

**Usage in agents:**

```python
SSM_GATEWAY_URL_PARAM   = os.environ.get('SSM_GATEWAY_URL_PARAM')
SSM_TREND_WORKLOAD_PARAM = os.environ.get('SSM_TREND_WORKLOAD_PARAM')

def get_oauth_config() -> dict:
    return {
        "gateway_url":    get_parameter_value(SSM_GATEWAY_URL_PARAM),
        "workload_name":  get_parameter_value(SSM_TREND_WORKLOAD_PARAM),
        "provider_name":  get_parameter_value(SSM_OAUTH_PROVIDER_PARAM),
        "scopes":         get_parameter_value(SSM_M2M_SCOPE_PARAM).split(","),
    }
```

---

## `env_generate.py` — UI `.env` File

The UI build requires a `.env` file with `VITE_*` keys pointing to deployed API endpoints and Cognito config. `env_generate.py` fetches all values from SSM and generates this file. Run it after `terraform apply` and before `npm run build`.

```python
# combined_terraform/env_generate.py (abbreviated)

import boto3
from pathlib import Path

def get_ssm_parameter(ssm_client, param_name, default=None):
    try:
        return ssm_client.get_parameter(Name=param_name, WithDecryption=True)['Parameter']['Value']
    except Exception:
        return default

def generate_env_file(project_name: str, region: str, output_path: str):
    session    = boto3.Session(region_name=region)
    ssm_client = session.client('ssm')

    # Fetch all values from SSM
    api_gateway_url  = get_ssm_parameter(ssm_client, f'/{project_name}/api/gateway-url', '')
    trend_agent_arn  = get_ssm_parameter(ssm_client, f'/{project_name}/agentcore/trend-agent-arn', '')
    anomaly_agent_arn = get_ssm_parameter(ssm_client, f'/{project_name}/agentcore/anomaly-agent-arn', '')

    # Build API endpoint URLs from base URL
    env_content = f"""# Auto-generated by env_generate.py — do not edit

VITE_TREND_AGENT_ARN={trend_agent_arn}
VITE_ANOMALY_AGENT_ARN={anomaly_agent_arn}

VITE_API_GET_PRODUCTS={api_gateway_url}/get-products
VITE_API_GET_SALES_SUMMARY={api_gateway_url}/get-sales-summary
VITE_API_START_REPORT={api_gateway_url}/start-sales-report
VITE_API_REPORT_STATUS={api_gateway_url}/get-report-status
VITE_API_GET_ANOMALIES={api_gateway_url}/get-anomalies
VITE_API_GET_TRENDS={api_gateway_url}/get-trend-analysis

VITE_USE_COGNITO_DOMAIN={get_ssm_parameter(ssm_client, f'/{project_name}/m2m/cognito_domain', '')}
VITE_USE_COGNITO_CLIENT_ID={get_ssm_parameter(ssm_client, f'/{project_name}/ui/client_id', '')}
"""
    Path(output_path).write_text(env_content)
```

Called by `deploy.sh`:
```bash
python3 combined_terraform/env_generate.py --project-name "${PROJECT_NAME}" --region "${AWS_REGION}"
```

The generated `.env` is written to `UI/.env` before `npm run build`.

---

## SecureString vs String

| Parameter type | Use for | Access |
|---|---|---|
| `String` | Non-sensitive values: bucket names, table names, URLs, ARNs | `get_parameter(Name=..., WithDecryption=False)` |
| `SecureString` | Sensitive values: API keys, tokens | `get_parameter(Name=..., WithDecryption=True)` |

**Pattern for secrets:** Store actual secrets in Secrets Manager, not SSM. Store the _Secrets Manager secret name_ in SSM as a `String`. This avoids writing secret values to Terraform state.

```hcl
# terraform/main.tf — store secret name reference in SSM
resource "aws_ssm_parameter" "m2m_secret_name" {
  name  = "/${var.project_name}/m2m/secret_name"
  type  = "String"
  value = aws_secretsmanager_secret.m2m_client_secret.name
}
```

```python
# deploy.sh — read the secret name from SSM, then fetch the actual secret
SECRET_NAME=$(aws ssm get-parameter --name "/${PROJECT_NAME}/m2m/secret_name" --query "Parameter.Value" --output text)
CLIENT_SECRET=$(aws secretsmanager get-secret-value --secret-id "$SECRET_NAME" --query "SecretString" --output text)
```

---

## Propagation Delay

After `terraform apply`, SSM parameters are available within seconds — but AgentCore containers may need a container restart to pick up new values (due to `lru_cache`). `deploy.sh` includes a 30-second wait:

```bash
echo "⏳ Waiting 30 seconds for SSM parameters to propagate..."
sleep 30
```

If a Lambda cold start fetches a stale cached SSM value, a `lambda:UpdateFunctionConfiguration` call (which forces a cold start) or waiting for the next cold start will refresh it.

---

## Viewing All Parameters for a Project

```bash
aws ssm get-parameters-by-path \
    --path "/${PROJECT_NAME}/" \
    --recursive \
    --query "Parameters[].{Name:Name,Value:Value}" \
    --output table \
    --region us-east-1
```

---

[← 08: Athena Views](08-athena-views-sql.md) &nbsp;·&nbsp; [🏠 Home](README.md) &nbsp;·&nbsp; [Next: 10 — Deployment Script →](10-deployment-script.md)
