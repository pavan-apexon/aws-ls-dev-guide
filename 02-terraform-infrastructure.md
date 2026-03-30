# 02 — Terraform Infrastructure Patterns

> **Quick reference:** [File-per-concern split](#file-per-concern-split) · [locals.tf](#localstf--computed-names) · [S3 policy-before-object](#s3-policy-before-object-race-condition) · [IAM pattern](#iam-roles--least-privilege) · [SSM config bus](#ssm-outputs-as-config-bus)

---

## File-per-concern Split

Split all resources across focused files in `terraform/`. One file = one concern. Never mix resource types, never put resources in `outputs.tf` or `variables.tf`.

```
terraform/
├── main.tf                    # Provider block + Auth (Cognito) + CloudFront/UI
├── versions.tf                # Required provider versions
├── variables.tf               # Input variable declarations only
├── locals.tf                  # Computed locals only (no resources)
├── outputs.tf                 # Output values only
├── data_infrastructure.tf     # S3 buckets, Glue, DynamoDB, Athena config
├── backend_lambdas.tf         # API-facing Lambda functions
├── agent_lambdas.tf           # Agent tool Lambda functions
├── report_lambda.tf           # Async report-generation Lambda trio + config
├── report_ssm_parameters.tf   # All SSM parameters for report runtime config
└── api_gateway.tf             # API Gateway: API, routes, integrations, CORS
```

This layout keeps `terraform plan` output readable and lets multiple developers work on different concerns without merge conflicts.

---

## `versions.tf` — Provider Constraints

```hcl
# terraform/versions.tf
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

---

## `variables.tf` — Input Variables

Define every externally configurable value with a description and sensible default. `project_name` is the single handle for the whole deployment.

```hcl
# terraform/variables.tf

variable "project_name" {
  description = "Project name prefix for all resources. Change to deploy a new environment. Example: retail-analytics-dev"
  type        = string
  default     = "retail-analytics-dev"
}

variable "env" {
  description = "Environment name (dev / qa / prod)"
  type        = string
  default     = "dev"
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "data_prefix" {
  description = "S3 key prefix where parquet data files are stored"
  type        = string
  default     = "data"
}

variable "lambda_runtime" {
  description = "Lambda runtime for all functions"
  type        = string
  default     = "python3.13"
}

variable "lambda_layer_arn" {
  description = "ARN of the Lambda layer (e.g. AWSSDKPandas)"
  type        = string
}

variable "agent_output_s3_prefix" {
  description = "S3 prefix for agent analysis output files"
  type        = string
  default     = "agent-outputs/"
}

variable "demo_username" {
  description = "Username for the Cognito demo user"
  type        = string
}

variable "demo_password" {
  description = "Password for the Cognito demo user"
  type        = string
  sensitive   = true
}
```

---

## `locals.tf` — Computed Names

Derive **all resource names** from `project_name` in one place. Never compute names inline inside resource blocks — that scatters logic and makes renaming error-prone.

```hcl
# terraform/locals.tf

locals {
  # Cognito
  user_pool_name  = "${var.project_name}-m2m-user-pool"
  m2m_client_name = "${var.project_name}-m2m-client"
  web_client_name = "${var.project_name}-web-client"

  # IAM roles
  lambda_execution_role_name = "${var.project_name}-lambda-execution-role"

  # S3 — append account ID for global uniqueness
  bucket_prefix = "${var.project_name}-${data.aws_caller_identity.current.account_id}"

  # DynamoDB
  traces_table_name = "${var.project_name}-agent-traces"

  # Glue — dashes are not valid in database names → replace with underscores
  glue_database_name = "${replace(var.project_name, "-", "_")}_database"
}
```

Reference locals throughout other `.tf` files:

```hcl
resource "aws_glue_catalog_database" "main" {
  name = local.glue_database_name   # always from locals, never hardcoded
}
```

---

## `main.tf` — Provider Block

The provider block lives only in `main.tf`. All other files inherit it.

```hcl
# terraform/main.tf
provider "aws" {
  region = var.region
}

data "aws_caller_identity" "current" {}
```

---

## S3 Buckets — Global Uniqueness

S3 bucket names are globally unique across ALL AWS accounts. Append the 12-digit account ID to guarantee no collision across teams or environments.

```hcl
# terraform/data_infrastructure.tf

locals {
  bucket_prefix = "${var.project_name}-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket" "data" {
  bucket        = "${local.bucket_prefix}-data"
  force_destroy = true

  tags = {
    Name        = "Data Bucket"
    Environment = var.env
    Project     = var.project_name
  }
}

resource "aws_s3_bucket" "athena_results" {
  bucket        = "${local.bucket_prefix}-athena-results"
  force_destroy = true
  tags = { Name = "Athena Results", Environment = var.env, Project = var.project_name }
}

resource "aws_s3_bucket" "agent_outputs" {
  bucket        = "${local.bucket_prefix}-agent-outputs"
  force_destroy = true
  tags = { Name = "Agent Outputs", Environment = var.env, Project = var.project_name }
}

resource "aws_s3_bucket" "model_artifacts" {
  bucket        = "${local.bucket_prefix}-model-artifacts"
  force_destroy = true
  tags = { Name = "Model Artifacts", Environment = var.env, Project = var.project_name }
}
```

---

## S3 Bucket Policies

Apply policies to restrict access to specific AWS service principals (Glue, Athena). Your IAM execution role still retains full access via IAM — the policy only governs service principals.

```hcl
# terraform/data_infrastructure.tf

resource "aws_s3_bucket_policy" "data_policy" {
  bucket = aws_s3_bucket.data.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowGlueAccess"
        Effect = "Allow"
        Principal = { Service = "glue.amazonaws.com" }
        Action   = ["s3:GetObject", "s3:ListBucket"]
        Resource = [aws_s3_bucket.data.arn, "${aws_s3_bucket.data.arn}/*"]
      },
      {
        Sid    = "AllowAthenaAccess"
        Effect = "Allow"
        Principal = { Service = "athena.amazonaws.com" }
        Action   = ["s3:GetObject", "s3:ListBucket"]
        Resource = [aws_s3_bucket.data.arn, "${aws_s3_bucket.data.arn}/*"]
      }
    ]
  })
}

resource "aws_s3_bucket_policy" "athena_results_policy" {
  bucket = aws_s3_bucket.athena_results.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowAthenaWrite"
        Effect = "Allow"
        Principal = { Service = "athena.amazonaws.com" }
        Action   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
        Resource = "${aws_s3_bucket.athena_results.arn}/*"
      },
      {
        Sid    = "AllowAthenaList"
        Effect = "Allow"
        Principal = { Service = "athena.amazonaws.com" }
        Action   = "s3:ListBucket"
        Resource = aws_s3_bucket.athena_results.arn
      }
    ]
  })
}
```

---

## S3 Policy-Before-Object Race Condition

**This is a known Terraform pitfall with the AWS provider ~> 5.0.**

When both a bucket policy and S3 objects declare `depends_on = [aws_s3_bucket.xxx]`, Terraform runs them **in parallel** the moment the bucket is ready. The policy application briefly makes the bucket return transient `NoSuchBucket` (not `AccessDenied`) when the provider calls `GetObjectTagging` post-upload to finalize state.

**Wrong — causes intermittent NoSuchBucket failures:**
```hcl
resource "aws_s3_object" "my_data" {
  bucket = aws_s3_bucket.data.id
  key    = "data/my_data.parquet"
  source = "../data/parquet/my_data.parquet"
  etag   = filemd5("../data/parquet/my_data.parquet")

  depends_on = [aws_s3_bucket.data]          # ← Only bucket — policy runs in parallel
}
```

**Correct — include the bucket policy in `depends_on`:**
```hcl
resource "aws_s3_object" "my_data" {
  bucket = aws_s3_bucket.data.id
  key    = "data/my_data.parquet"
  source = "../data/parquet/my_data.parquet"
  etag   = filemd5("../data/parquet/my_data.parquet")

  depends_on = [
    aws_s3_bucket.data,
    aws_s3_bucket_policy.data_policy         # ← Wait for policy to fully apply first
  ]

  tags = { DataType = "Transactions" }
}
```

Apply this pattern to **every** `aws_s3_object` that uploads into a bucket with a bucket policy.

---

## IAM Roles — Least Privilege

Use a three-part structure for every Lambda IAM role: trust policy, basic execution managed policy, and a custom inline policy scoped to exactly what the function needs.

```hcl
# terraform/agent_lambdas.tf

# Part 1: trust policy
resource "aws_iam_role" "agent_lambda_role" {
  name = "${var.project_name}-agent-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })

  tags = { Environment = var.env, Project = var.project_name }
}

# Part 2: CloudWatch Logs (always needed)
resource "aws_iam_role_policy_attachment" "agent_lambda_basic" {
  role       = aws_iam_role.agent_lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Part 3: only the permissions this function actually uses
resource "aws_iam_role_policy" "agent_lambda_policy" {
  name = "${var.project_name}-agent-lambda-policy"
  role = aws_iam_role.agent_lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "athena:StartQueryExecution", "athena:GetQueryExecution",
          "athena:GetQueryResults", "glue:GetTable", "glue:GetDatabase"
        ]
        Resource = "*"
      },
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:PutObject"]
        Resource = "${aws_s3_bucket.data.arn}/*"
      },
      {
        Effect   = "Allow"
        Action   = ["dynamodb:PutItem", "dynamodb:GetItem", "dynamodb:Query", "dynamodb:UpdateItem"]
        Resource = aws_dynamodb_table.agent_results.arn
      },
      {
        # Scope to this project's parameters only — not all of SSM
        Effect   = "Allow"
        Action   = ["ssm:GetParameter", "ssm:GetParameters"]
        Resource = "arn:aws:ssm:${var.region}:${data.aws_caller_identity.current.account_id}:parameter/${var.project_name}/*"
      }
    ]
  })
}
```

---

## DynamoDB Tables

Use `PAY_PER_REQUEST` for all tables. Enable PITR and SSE on tables with business data. Add TTL on tables storing transient state (async jobs, session data).

```hcl
# terraform/data_infrastructure.tf

# Business data table — PITR + encryption
resource "aws_dynamodb_table" "report_results" {
  name         = "${var.project_name}-report-results"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "PK"
  range_key    = "SK"

  attribute { name = "PK", type = "S" }
  attribute { name = "SK", type = "S" }

  point_in_time_recovery { enabled = true }
  server_side_encryption  { enabled = true }

  tags = { Name = "Report Results", Environment = var.env, Project = var.project_name }
}

# Transient job tracking table — TTL to auto-expire records after 24 h
resource "aws_dynamodb_table" "async_jobs" {
  name         = "${var.project_name}-async-jobs"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "job_id"

  attribute { name = "job_id", type = "S" }

  ttl {
    attribute_name = "expires_at"   # Lambda sets this to epoch_now + 86400
    enabled        = true
  }

  tags = { Name = "Async Jobs", Environment = var.env, Project = var.project_name }
}
```

---

## SSM Outputs as Config Bus

Terraform writes every runtime-relevant value to SSM Parameter Store immediately after creating the resource. Lambdas, agents, and the UI env generator all read from SSM — no values are passed through hard-coded environment variables set at deploy time.

### Naming convention: `/{project_name}/{service}/{param}`

```hcl
# terraform/report_ssm_parameters.tf

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

resource "aws_ssm_parameter" "athena_output_location" {
  name        = "/${var.project_name}/athena/output-location"
  type        = "String"
  value       = "s3://${aws_s3_bucket.athena_results.bucket}/"
  description = "S3 location for Athena query results"
  tags        = { Environment = var.env, Project = var.project_name }
}

# Agent ARNs are written here by deploy.sh after `agentcore launch`
# Terraform creates the placeholder; the shell script overwrites it with the real ARN
resource "aws_ssm_parameter" "trend_agent_arn" {
  name        = "/${var.project_name}/agentcore/trend-agent-arn"
  type        = "String"
  value       = "PLACEHOLDER"
  description = "ARN of the deployed trend analysis AgentCore agent"
  tags        = { Environment = var.env, Project = var.project_name }
}
```

---

## `outputs.tf` — What to Export

Export values that downstream consumers need: `deploy.sh`, `env_generate.py`, other modules, or CI pipelines.

```hcl
# terraform/outputs.tf

output "project_name" {
  description = "Project name — used by deploy.sh to derive agent names and SSM paths"
  value       = var.project_name
}

output "data_bucket_name" {
  value = aws_s3_bucket.data.bucket
}

output "api_gateway_url" {
  description = "Base URL for all API calls from the UI"
  value       = aws_apigatewayv2_api.main.api_endpoint
}

output "cloudfront_url" {
  value = "https://${aws_cloudfront_distribution.ui.domain_name}"
}

output "cognito_user_pool_id" {
  value = aws_cognito_user_pool.m2m_pool.id
}

output "m2m_client_id" {
  value = aws_cognito_user_pool_client.m2m_client.id
}

output "m2m_client_secret" {
  value     = aws_cognito_user_pool_client.m2m_client.client_secret
  sensitive = true
}
```

---

[← 01: Project Structure](01-project-structure.md) &nbsp;·&nbsp; [🏠 Home](README.md) &nbsp;·&nbsp; [Next: 03 — Data Layer →](03-data-layer.md)
