# AWS Analytics Platform — Developer Reference Guide

A practical, example-driven reference for building AWS-native analytics applications. Every pattern shown is built around a simple, universally-understood example — a **retail sales analytics platform** — so no domain expertise is required to follow along.

> **Audience:** Tiered — quick-start pointers at the top of each file for experienced AWS developers; deep-dive sections for developers new to this stack.

---

## What is an AWS Analytics Application?

An AWS analytics application is a full-stack system built on AWS that combines:

- **Data Infrastructure** — S3 + Glue + Athena for storing and querying tabular data (orders, products, customers, discounts)
- **Intelligent Agents** — Bedrock AgentCore agents (Strands SDK) for automated trend analysis and anomaly detection
- **API Layer** — API Gateway + Lambda functions exposing data to the frontend
- **Async Job Processing** — A three-Lambda pattern for operations that exceed API Gateway's 29-second integration timeout
- **Frontend** — React/TypeScript + AWS Cloudscape UI with OAuth2/Cognito authentication
- **Infrastructure as Code** — Terraform managing all resources with a single `project_name` variable flowing through everything

---

## Example Project: `retail-analytics`

Throughout this guide, all examples use a **retail sales analytics** project. This project:

- Tracks sales orders across products, customers, regions, and channels
- Stores 9 source tables in S3 (orders, products, customers, order_discounts, contracts, suppliers, distributors, regions, regional_tax_rates)
- Exposes API endpoints for product lists, sales summaries, and report generation
- Runs two AI agents: a **trend analysis agent** and an **anomaly detection agent**
- Generates complex cross-region sales reports (a long-running operation that triggers the async pattern)

You can substitute any other domain — the AWS architecture patterns are identical regardless of what data you store.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DEVELOPER MACHINE                           │
│   terraform apply  ──►  python gateway.py  ──►  agentcore launch   │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  provisions / deploys
                    ┌────────────▼────────────┐
                    │        AWS Cloud         │
                    └────────────┬────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
   ┌──────▼──────┐       ┌───────▼──────┐       ┌──────▼──────┐
   │  Data Layer │       │  Agent Layer │       │   UI Layer  │
   │             │       │              │       │             │
   │  S3 Parquet │       │  AgentCore   │       │  CloudFront │
   │  Glue Catalog       │  Agents      │       │  + S3       │
   │  Athena Views       │  (Strands)   │       │             │
   │  DynamoDB   │       │  MCP Gateway │       │  Cognito    │
   └──────┬──────┘       └───────┬──────┘       │  OAuth2     │
          │                      │               └──────┬──────┘
          └──────────────────────▼─────────────────────┘
                         ┌───────────────┐
                         │  API Gateway  │
                         │  + Lambda fns │
                         └───────────────┘
```

### Request flow (end-to-end)
1. User opens React UI → authenticated via Cognito (OAuth2 / SSO)
2. UI calls API Gateway → routes to a backend Lambda function
3. Backend Lambda queries Athena (analytical views) or DynamoDB (report results / agent outputs)
4. **For long-running operations:** Initiator Lambda returns `job_id` immediately → Worker Lambda runs the computation (60–120 s) → Status Checker Lambda polls for result
5. **For analysis:** Bedrock AgentCore agent invoked via MCP Gateway → agent calls Lambda tools → result stored in DynamoDB/S3
6. All runtime config flows through SSM Parameter Store — nothing is hardcoded after deployment

---

## Pre-requisites

| Tool | Min version | Purpose |
|------|-------------|---------|
| Terraform | >= 1.0 | Infrastructure provisioning |
| Python | >= 3.11 | Post-deploy scripts, Lambda/agent runtimes |
| Node.js + npm | LTS | UI build |
| AWS CLI | v2 | Resource management, SSM reads |
| agentcore CLI | latest | Bedrock AgentCore agent build & deploy |
| boto3 | >= 1.34 | Python AWS SDK (used in all scripts) |

**AWS permissions required:** IAM role or user with access to S3, Lambda, API Gateway, Glue, Athena, DynamoDB, Bedrock AgentCore, Cognito, CloudFront, SSM Parameter Store, Secrets Manager, IAM, ECR.

---

## Quick Start

```bash
# 1. Set the one variable that drives the whole deployment
PROJECT_NAME="retail-analytics-dev"

# 2. Deploy all AWS infrastructure
cd terraform/
terraform init
terraform apply -var="project_name=${PROJECT_NAME}"

# 3. Wait for SSM parameters to propagate, then run post-deploy scripts
sleep 30
export PROJECT_NAME AWS_REGION="us-east-1"
python agentcore_gateway/gateway.py
python agentcore_identity/create_workload_identity.py

# 4. Deploy agents (repeat for each agent)
cd agents/trend-analysis-agent/agent/
agentcore configure \
  --name "${PROJECT_NAME//-/_}_trend_analysis_agent" \
  --entrypoint trend_analysis_agent.py \
  --execution-role $(aws ssm get-parameter \
      --name /${PROJECT_NAME}/bedrock-agentcore/role-name \
      --query Parameter.Value --output text)
agentcore launch --env "PROJECT_NAME=${PROJECT_NAME}" --env "AWS_REGION=us-east-1"

# 5. Create Athena views
cd terraform/sql/
aws athena start-query-execution \
  --query-string "$(cat monthly_sales_view.sql)" \
  --result-configuration OutputLocation=$(aws ssm get-parameter \
      --name /${PROJECT_NAME}/athena/output-location --query Parameter.Value --output text) \
  --query-execution-context Database=$(aws ssm get-parameter \
      --name /${PROJECT_NAME}/data/athena-database --query Parameter.Value --output text)

# 6. Build and deploy UI
python env_generate.py --project-name ${PROJECT_NAME} --region us-east-1
cd UI/ && npm install && npm run build
aws s3 sync dist/ s3://$(aws ssm get-parameter \
    --name /${PROJECT_NAME}/ui/s3-bucket-name \
    --query Parameter.Value --output text)/ --delete

# — OR — run everything with the single deployment script:
bash deploy.sh
```

---

## Guide Index

| # | Guide | What it covers |
|---|-------|----------------|
| 1 | [Project Structure & Conventions](./01-project-structure.md) | Canonical folder layout, naming rules, what goes where |
| 2 | [Terraform Infrastructure Patterns](./02-terraform-infrastructure.md) | File-per-concern, `locals.tf`, SSM outputs, IAM, S3 policies |
| 3 | [Data Layer — S3 + Glue + Parquet](./03-data-layer.md) | Data ingestion, Glue catalog tables, Athena configuration |
| 4 | [Backend Lambda API Pattern](./04-backend-lambda-apis.md) | Lambda handler, Athena query, DynamoDB read, API Gateway wiring |
| 5 | [Async Lambda Architecture](./05-async-lambda-architecture.md) | Initiator → Worker → StatusChecker for operations exceeding API Gateway's 29-second limit |
| 6 | [Bedrock Agent Development (Strands)](./06-bedrock-agents-strands.md) | Agent folder layout, Strands SDK, MCP tools, runtime config |
| 7 | [AgentCore Gateway & Workload Identity](./07-agentcore-gateway-identity.md) | Idempotent gateway setup, OAuth authorizer, workload identity |
| 8 | [Athena Views & SQL Management](./08-athena-views-sql.md) | SQL view files, CLI execution, dependency sequencing |
| 9 | [Dynamic Config via SSM Parameter Store](./09-dynamic-config-ssm.md) | SSM naming convention, Terraform writes, Lambda reads, env generation |
| 10 | [Full Deployment Script](./10-deployment-script.md) | Phase-by-phase deploy.sh walkthrough — infra → agents → UI |
| 11 | [React / TypeScript UI Patterns](./11-ui-react-typescript.md) | Cloudscape components, Bedrock client, OAuth flow, S3+CloudFront hosting |

---

## Key Design Principles

### 1. Single source of truth: `project_name`
Everything — S3 buckets, DynamoDB tables, IAM roles, SSM parameters, Cognito pools, agent names — is derived from one `project_name` Terraform variable. Changing it redeploys the entire stack under a new name with zero code changes.

### 2. SSM Parameter Store as the config bus
Terraform writes every runtime value (ARNs, bucket names, table names, endpoints) to SSM at creation time. Lambdas, agents, and UI env-generation all read from SSM at runtime. Nothing is hardcoded post-deployment.

### 3. Agents are code, not config
Each Bedrock agent lives in its own folder (`agent/`, `tools/`, `schemas/`), is containerised by AgentCore, and communicates with Lambda tools through the MCP Gateway. The gateway and tool schemas are registered programmatically via `gateway.py`.

### 4. `depends_on` for policy-before-object
S3 bucket objects must explicitly depend on the bucket policy, not just the bucket. Without this, Terraform uploads objects in parallel with policy application, causing transient `NoSuchBucket` errors on `GetObjectTagging`. See [Terraform Patterns](./02-terraform-infrastructure.md#s3-policy-before-object-race-condition).

### 5. Async for all long-running operations
API Gateway has a hard **29-second integration timeout**. Any Lambda that runs longer than 29 seconds — report generation, bulk data export, complex multi-table computation — must use the Initiator → Worker → StatusChecker pattern with DynamoDB job tracking. See [Async Lambda Architecture](./05-async-lambda-architecture.md).

### 6. `boto3` for list/query operations on AgentCore, `GatewayClient` for create operations
The `bedrock_agentcore_starter_toolkit.GatewayClient` does not expose `list_gateways()`. Use `boto3.client('bedrock-agentcore-control')` for all list/query operations and `GatewayClient` only for create/manage. See [Gateway Setup](./07-agentcore-gateway-identity.md).

---

[🏠 Home](README.md) &nbsp;·&nbsp; [Next: 01 — Project Structure →](01-project-structure.md)

---

## About This Guide

**Author:** Pavan Gandhi

This guide is an **initial draft** written to give developers a structured, end-to-end reference for building on this AWS analytics platform. It covers infrastructure, data, async patterns, Bedrock agents, AgentCore, Athena, SSM config, deployment, and UI — all in one place.

It is intentionally kept as close to the real codebase as possible, but it is a living document. Things will drift as the project evolves, and sections written by one person may not fully reflect decisions made elsewhere (for example, the UI section carries an explicit disclaimer).

**You are encouraged to:**
- Fix anything that is inaccurate or out of date
- Add sections for patterns not yet covered
- Improve explanations where they are unclear
- Extend examples as the project grows

**To contribute:** raise a pull request against this repo. Even small corrections — a typo fix, a clarification, an updated SSM path — are welcome.

**To reach the author** for context, questions, or to hand off ownership of this guide, contact **Pavan Gandhi** directly.
