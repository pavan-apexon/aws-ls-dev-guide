# 10 — Deployment Script

> **Quick reference:** [Phase overview](#phase-overview) · [deploy_infra](#phase-1-deploy_infra) · [deploy_agent](#phase-2-deploy_agent) · [Athena views](#phase-3-create_athena_views) · [UI build/deploy](#phase-4-build-and-deploy-ui) · [Re-deploy patterns](#re-deploy-patterns) · [Troubleshooting](#troubleshooting)

---

## Overview

`deploy.sh` is the **single entrypoint** for a fresh deployment or re-deployment. It orchestrates five phases in order:

```
1. deploy_infra         → terraform apply + post-Terraform Python scripts
2. create_athena_views  → submit SQL view files to Athena
3. deploy_agent (Trend) → agentcore configure + deploy + store ARN in SSM
4. deploy_agent (Anomaly) → same for anomaly detection agent
5. build_ui + deploy_ui → npm run build + S3 sync + CloudFront invalidation
```

---

## Prerequisites

```bash
# Tools required on the machine running deploy.sh
terraform  --version   # >= 1.0
aws        --version   # configured with sufficient permissions
agentcore  --version   # installed: pip install bedrock-agentcore-starter-toolkit
python3    --version   # >= 3.11
node       --version   # >= 18 (for UI build)
npm        --version   # >= 9
docker              # running — agentcore uses it to build container images
```

---

## Running a Full Deploy

```bash
# First deploy
bash deploy.sh

# Re-deploy everything (idempotent — safe to run repeatedly)
bash deploy.sh
```

All phases are idempotent: Terraform only changes what has drifted, gateway setup skips existing targets, `agentcore deploy` updates in place, S3 sync uploads only changed files.

---

## Phase 1: `deploy_infra`

```bash
deploy_infra() {
    # 1. Run terraform apply
    pushd combined_terraform >/dev/null
    terraform init -upgrade
    terraform apply
    popd >/dev/null

    # 2. Read project_name from terraform output
    PROJECT_NAME=$(cd combined_terraform && terraform output -raw project_name)

    # 3. Derive agent names (hyphens → underscores for Python module names)
    AGENT_TREND_NAME="$(echo $PROJECT_NAME | tr '-' '_')_trend_analysis_agent"
    AGENT_ANOMALY_NAME="$(echo $PROJECT_NAME | tr '-' '_')_anomaly_detection_agent"

    # 4. Wait for SSM propagation, then run post-Terraform scripts
    sleep 30
    python3 combined_terraform/Agentcore_gateway/gateway.py
    python3 combined_terraform/Agentcore_identity/create_workload_identity.py

    # 5. Create OAuth2 credential provider
    create_oauth_provider
}
```

**Key points:**
- `terraform output -raw project_name` — the single source of truth for the project name. All downstream steps derive everything from this value.
- `tr '-' '_'` — AgentCore agent names must use underscores, not hyphens (Python identifier rules). Always apply this transform when setting `$AGENT_TREND_NAME`.
- `sleep 30` — SSM parameters need ~30 seconds to propagate globally before the Python scripts read them.

---

## `fetch_param` Helper

```bash
fetch_param() {
    aws ssm get-parameter \
        --name "$1" \
        --query "Parameter.Value" \
        --output text \
        --region "${AWS_REGION}" 2>/dev/null || echo ""
}

# Usage
API_URL=$(fetch_param "/${PROJECT_NAME}/api/gateway-url")
```

The `2>/dev/null || echo ""` makes it return an empty string if the parameter doesn't exist, rather than failing the script (useful for optional parameters).

---

## `create_oauth_provider`

```bash
create_oauth_provider() {
    PROVIDER_NAME=$(aws ssm get-parameter --name "/${PROJECT_NAME}/m2m/oauth_provider_name" ...)
    ISSUER_URL=$(aws ssm get-parameter --name "/${PROJECT_NAME}/m2m/discovery_url" ...)
    CLIENT_ID=$(aws ssm get-parameter --name "/${PROJECT_NAME}/m2m/client_id" ...)

    # Secret lives in Secrets Manager — never in SSM
    SECRET_NAME=$(aws ssm get-parameter --name "/${PROJECT_NAME}/m2m/secret_name" ...)
    CLIENT_SECRET=$(aws secretsmanager get-secret-value --secret-id "$SECRET_NAME" --query "SecretString" --output text)

    # Write a temp JSON config file (never echo credentials to the terminal)
    cat > oauth_config_temp.json <<EOF
{
  "customOauth2ProviderConfig": {
    "oauthDiscovery": { "discoveryUrl": "${ISSUER_URL}" },
    "clientId":     "${CLIENT_ID}",
    "clientSecret": "${CLIENT_SECRET}"
  }
}
EOF

    aws bedrock-agentcore-control create-oauth2-credential-provider \
        --name "${PROVIDER_NAME}" \
        --credential-provider-vendor "CustomOauth2" \
        --oauth2-provider-config-input "file://oauth_config_temp.json"

    rm -f oauth_config_temp.json      # ← Always clean up the secret file
}
```

---

## Phase 2: `deploy_agent`

The `deploy_agent` function handles both agents through a parameterized interface:

```bash
# deploy.sh — main section
deploy_agent "${AGENT_TREND_NAME}" "${TREND_DIR}" "${TREND_ENTRY}" auto true \
    "/${PROJECT_NAME}/agentcore/trend-agent-arn" \
    "SSM_OAUTH_PROVIDER_PARAM=/${PROJECT_NAME}/m2m/oauth_provider_name" \
    "SSM_TREND_WORKLOAD_PARAM=/${PROJECT_NAME}/agentcore/trend-workload-name" \
    "SSM_M2M_SCOPE_PARAM=/${PROJECT_NAME}/m2m/scope" \
    "SSM_GATEWAY_URL_PARAM=/${PROJECT_NAME}/gateway/url" \
    "SSM_AGENT_TRACES_TABLE_PARAM=/${PROJECT_NAME}/dynamodb/agent-traces-table"
```

**Inside `deploy_agent`:**

```bash
deploy_agent() {
    local AGENT_NAME=$1       # e.g. retail_analytics_dev_trend_analysis_agent
    local AGENT_DIR=$2        # path to agent folder with .bedrock_agentcore.yaml
    local ENTRYPOINT=$3       # Python file with BedrockAgentCoreApp
    local AGENT_ECR=${4:-auto}
    local ENABLE_OAUTH=${5:-false}
    local SSM_PARAM_NAME=$6   # where to store the agent ARN after deploy

    pushd "${AGENT_DIR}" >/dev/null

    # Step 1: Configure (update .bedrock_agentcore.yaml + create ECR repo)
    agentcore configure \
        --non-interactive \
        --name "${AGENT_NAME}" \
        --entrypoint "${ENTRYPOINT}" \
        --execution-role "${AWS_ROLE_NAME}" \
        --ecr "${AGENT_ECR}" \
        --region "${AWS_REGION}" \
        --authorizer-config "${OAUTH_JSON}"       # only if ENABLE_OAUTH=true

    # Step 2: Deploy (build Docker image, push to ECR, update agent runtime)
    agentcore deploy \
        --auto-update-on-conflict \
        --env "PROJECT_NAME=${PROJECT_NAME}" \
        --env "AWS_REGION=${AWS_REGION}" \
        --env "SSM_OAUTH_PROVIDER_PARAM=/${PROJECT_NAME}/m2m/oauth_provider_name" \
        # ... additional SSM parameter name env vars

    # Step 3: Store the deployed ARN in SSM
    sleep 5
    AGENT_ARN=$(aws bedrock-agentcore-control list-agent-runtimes \
        --query "agentRuntimes[?agentRuntimeName=='${AGENT_NAME}'].agentRuntimeArn" \
        --output text)

    aws ssm put-parameter \
        --name "${SSM_PARAM_NAME}" \
        --value "${AGENT_ARN}" \
        --type "String" \
        --overwrite

    popd >/dev/null
}
```

**The `--env` flags pass SSM parameter names** (not values) into the container as environment variables. The agent code reads `os.environ.get('SSM_GATEWAY_URL_PARAM')` to know _which_ SSM key to fetch.

---

## Phase 3: `create_athena_views`

```bash
create_athena_views() {
    DATABASE_NAME=$(fetch_param "/${PROJECT_NAME}/data/athena-database")
    ATHENA_OUTPUT=$(fetch_param  "/${PROJECT_NAME}/athena/output-location")

    pushd combined_terraform >/dev/null

    execute_query() {
        aws athena start-query-execution \
            --query-string "$(cat $1)" \
            --result-configuration OutputLocation="${ATHENA_OUTPUT}" \
            --query-execution-context Database="${DATABASE_NAME}" \
            --region "${AWS_REGION}"
    }

    # Order matters — base views first, then dependent views
    execute_query "sql/monthly_sales_view.sql"
    execute_query "sql/quarterly_revenue_view.sql"
    execute_query "sql/channel_performance_view.sql"
    execute_query "sql/region_performance_view.sql"
    execute_query "sql/customer_segment_view.sql"

    sleep 5

    execute_query "sql/anomaly_detection_view.sql"
    execute_query "sql/trend_analysis_view.sql"

    popd >/dev/null
}
```

The `sleep` calls ensure Athena finishes registering each view before the next dependent view references it. Without them, views that reference other views may fail with "table not found."

---

## Phase 4: Build and Deploy UI

```bash
build_ui() {
    pushd UI >/dev/null
    npm install
    npm run build           # outputs to UI/dist/
    popd >/dev/null
}

deploy_ui() {
    UI_BUCKET=$(fetch_param "/${PROJECT_NAME}/ui/s3-bucket-name")
    CLOUDFRONT_ID=$(fetch_param "/${PROJECT_NAME}/ui/cloudfront-distribution-id")

    # Sync built assets — delete files in S3 that no longer exist locally
    aws s3 sync UI/dist "s3://${UI_BUCKET}/" --delete

    # Invalidate CloudFront cache so users get the new version immediately
    aws cloudfront create-invalidation \
        --distribution-id "${CLOUDFRONT_ID}" \
        --paths "/*"

    # Update Cognito callback URLs with the actual CloudFront domain
    CLOUDFRONT_URL=$(aws cloudfront get-distribution \
        --id "$CLOUDFRONT_ID" \
        --query "Distribution.DomainName" --output text)

    aws cognito-idp update-user-pool-client \
        --user-pool-id "${USER_POOL_ID}" \
        --client-id "${APP_CLIENT_ID}" \
        --callback-urls "https://${CLOUDFRONT_URL}/sso-login" "http://localhost:5173" \
        --logout-urls "https://${CLOUDFRONT_URL}/sso-login" "http://localhost:5173" \
        # ...
}
```

The `--delete` flag in `aws s3 sync` removes stale assets. Include it — without it, old versioned bundles accumulate in S3.

---

## Re-Deploy Patterns

| Scenario | Command |
|---|---|
| First fresh deploy | `bash deploy.sh` |
| Terraform-only change | `cd combined_terraform && terraform apply` |
| Agent code change (Trend)   | `cd agents/trend-analysis-agent && agentcore deploy --auto-update-on-conflict` |
| Agent code change (Anomaly) | `cd agents/anomaly-detection-agent && agentcore deploy --auto-update-on-conflict` |
| UI change only | `cd UI && npm run build` then `aws s3 sync UI/dist s3://{bucket}/ --delete` + invalidate |
| Add a new gateway target | Edit `gateway.py`, run `python3 combined_terraform/Agentcore_gateway/gateway.py` |
| Regenerate UI .env | `python3 combined_terraform/env_generate.py --project-name retail-analytics-dev --region us-east-1` |

---

## Troubleshooting

**`terraform apply` fails with "existing resource" errors:**
```bash
terraform state list   # Check what's already in state
terraform import aws_resource.name resource-id   # Import if missing from state
```

**`agentcore configure` fails with "ECR repo already exists":**
Use `--ecr auto` (not `--ecr my-project-dev-rca-agent`) — `auto` re-uses the existing repo.

**Agent ARN not stored in SSM after deploy:**
The `sleep 5` before `list-agent-runtimes` may not be enough. Run manually:
```bash
aws bedrock-agentcore-control list-agent-runtimes \
    --query "agentRuntimes[?agentRuntimeName=='${AGENT_NAME}'].agentRuntimeArn" \
    --output text
aws ssm put-parameter --name "/${PROJECT_NAME}/agentcore/trend-agent-arn" \
    --value "${ARN}" --type String --overwrite
```

**Athena view creation fails with "Table not found":**
The view it references was not yet created. Increase `sleep` values between view groups, or create the dependency view manually first via the Athena console.

**UI shows stale data after deploy:**
CloudFront invalidation takes 1–3 minutes. If the cache persists, run:
```bash
aws cloudfront create-invalidation --distribution-id ${CLOUDFRONT_ID} --paths "/*"
```

---

[← 09: Dynamic Config (SSM)](09-dynamic-config-ssm.md) &nbsp;·&nbsp; [🏠 Home](README.md) &nbsp;·&nbsp; [Next: 11 — UI →](11-ui-react-typescript.md)
