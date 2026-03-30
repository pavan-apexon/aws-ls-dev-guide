# 07 — AgentCore Gateway and Workload Identity

> **Quick reference:** [What the gateway does](#what-the-mcp-gateway-does) · [GatewayClient vs boto3](#gatewayclient-vs-boto3) · [Idempotent setup](#idempotent-gateway-setup) · [Tool targets](#tool-targets) · [OAuth provider](#oauth-provider) · [Workload identity](#workload-identity) · [Token flow](#token-flow)

---

## What the MCP Gateway Does

The MCP (Model Context Protocol) gateway is a managed endpoint that bridges the Strands agent to the Lambda tool functions. Without it, each agent would need VPC access or direct Lambda ARNs to call tools — with it, tool calls go over authenticated HTTPS.

```
Agent container
    ↓  HTTP POST (Bearer token)
MCP Gateway (AgentCore-managed URL)
    ↓  Invokes by Lambda ARN
Lambda tool functions
    ↓  Returns JSON
MCP Gateway
    ↓  Returns JSON response
Agent container
```

The gateway handles:
- OAuth JWT validation on incoming requests
- Lambda invocation routing
- Tool schema registration for semantic search

---

## GatewayClient vs boto3 — Critical Distinction

**Always use the right client for the right operation.** Mixing them causes `AttributeError` at runtime.

| Operation | Use | Reason |
|---|---|---|
| `create_mcp_gateway()` | `GatewayClient` (toolkit) | Handled by bedrock_agentcore_starter_toolkit |
| `create_mcp_gateway_target()` | `GatewayClient` (toolkit) | Same |
| `fix_iam_permissions()` | `GatewayClient` (toolkit) | Same |
| `list_mcp_gateway_targets()` | `GatewayClient` (toolkit) | Same |
| `list_gateways()` | `boto3.client('bedrock-agentcore-control')` | Toolkit does NOT expose list operations |
| `list_agent_runtimes()` | `boto3.client('bedrock-agentcore-control')` | Same — any list/query op |

```python
from bedrock_agentcore_starter_toolkit.operations.gateway.client import GatewayClient
import boto3

# For create/manage operations
client = GatewayClient(region_name=region)

# For list/query operations — toolkit has no list_gateways()
agentcore_control = boto3.client('bedrock-agentcore-control', region_name=region)
```

---

## Idempotent Gateway Setup

The gateway setup script must be runnable multiple times safely (Terraform re-runs `deploy.sh` on every deploy).

The pattern:
1. List existing gateways with the boto3 control client.
2. Check by name (case-insensitive).
3. If gateway exists: check for missing targets and add them — then return.
4. If gateway does not exist: create it, then add all targets.

```python
# combined_terraform/Agentcore_gateway/gateway.py (abbreviated)

from bedrock_agentcore_starter_toolkit.operations.gateway.client import GatewayClient
import boto3, json, os

def setup_gateway():
    region       = os.environ.get('AWS_REGION', 'us-east-1')
    project_name = os.environ['PROJECT_NAME']

    ssm_client         = boto3.client('ssm', region_name=region)
    client             = GatewayClient(region_name=region)
    agentcore_control  = boto3.client('bedrock-agentcore-control', region_name=region)

    # Build gateway name from project name
    gateway_name = f"{project_name.title().replace('-', '')}-analytics-gateway"

    # Fetch required config from SSM
    gateway_role_arn = ssm_client.get_parameter(Name=f"/{project_name}/gateway/role-arn")['Parameter']['Value']
    m2m_client_id    = ssm_client.get_parameter(Name=f"/{project_name}/m2m/client_id")['Parameter']['Value']
    web_client_id    = ssm_client.get_parameter(Name=f"/{project_name}/ui/client_id")['Parameter']['Value']
    discovery_url    = ssm_client.get_parameter(Name=f"/{project_name}/m2m/discovery_url")['Parameter']['Value']

    # OAuth authorizer config
    authorizer_config = {
        "customJWTAuthorizer": {
            "allowedClients": [m2m_client_id, web_client_id],
            "discoveryUrl":   discovery_url,
        }
    }

    # ── Check for existing gateway ──────────────────────────────────
    existing = agentcore_control.list_gateways()
    gateway_list = existing.get('items', existing.get('gateways', []))

    # Case-insensitive name match
    gateway_exists = any(
        g.get('name', '').lower() == gateway_name.lower()
        for g in gateway_list
    )

    if gateway_exists:
        # Already exists — find it and add any missing targets
        gateway = next(g for g in gateway_list if g['name'].lower() == gateway_name.lower())
        add_missing_targets(client, ssm_client, gateway, project_name)
    else:
        # Create and register all targets
        gateway = client.create_mcp_gateway(
            name=gateway_name,
            role_arn=gateway_role_arn,
            authorizer_config=authorizer_config,
            enable_semantic_search=True,
        )
        client.fix_iam_permissions(gateway)
        import time; time.sleep(30)      # Wait for IAM propagation
        add_all_targets(client, ssm_client, gateway, project_name)

    # Always write gateway URL and ID to SSM (idempotent overwrite)
    ssm_client.put_parameter(
        Name=f"/{project_name}/gateway/url",
        Value=gateway.get('gatewayUrl', ''),
        Type='String', Overwrite=True,
    )
```

---

## Tool Targets

Each tool target maps a Lambda function to a named tool endpoint inside the gateway. The gateway uses the JSON schema to validate tool calls from the agent.

```python
lambda_targets = [
    {
        "param_key": f"/{project_name}/lambda/get-sales-by-region-arn",
        "name":      "get-sales-by-region",
        "schema":    "get-sales-by-region.json",
    },
    {
        "param_key": f"/{project_name}/lambda/get-trend-metrics-arn",
        "name":      "get-trend-metrics",
        "schema":    "get-trend-metrics.json",
    },
    # ... one entry per Lambda tool function
]

def add_all_targets(client, ssm_client, gateway, project_name):
    for target in lambda_targets:
        lambda_arn = ssm_client.get_parameter(Name=target["param_key"])['Parameter']['Value']

        schema_path = os.path.join(os.path.dirname(__file__), "schemas", target["schema"])
        with open(schema_path) as f:
            tool_schema = json.load(f)

        client.create_mcp_gateway_target(
            gateway=gateway,
            name=target["name"],
            target_type="lambda",
            target_payload={
                "lambdaArn":  lambda_arn,
                "toolSchema": {
                    "inlinePayload": [tool_schema[0]] if isinstance(tool_schema, list) else [tool_schema]
                },
            },
            credentials=None,
        )
        print(f"✓ Added target: {target['name']}")
```

---

## Tool Schema Format

Each `schemas/{name}.json` file follows the JSON Schema format. The gateway uses it to validate parameters passed by the agent.

```json
[
  {
    "name": "get-sales-by-region",
    "description": "Fetch aggregated sales totals for a product broken down by region and time period.",
    "inputSchema": {
      "json": {
        "type": "object",
        "properties": {
          "product_id": { "type": "string",  "description": "Product unique identifier" },
          "year":       { "type": "integer", "description": "Year (2020–2030)" },
          "month":      { "type": "integer", "description": "Month (1–12)" }
        },
        "required": ["product_id", "year", "month"]
      }
    }
  }
]
```

---

## OAuth Provider

The OAuth provider tells AgentCore which Cognito user pool to validate tokens against. Created once via the AWS CLI (wrapped in `deploy.sh`):

```bash
# deploy.sh — called after terraform apply

PROVIDER_NAME=$(aws ssm get-parameter --name "/${PROJECT_NAME}/m2m/oauth_provider_name" --query "Parameter.Value" --output text)
ISSUER_URL=$(aws ssm get-parameter --name "/${PROJECT_NAME}/m2m/discovery_url" --query "Parameter.Value" --output text)
CLIENT_ID=$(aws ssm get-parameter --name "/${PROJECT_NAME}/m2m/client_id" --query "Parameter.Value" --output text)

# Get client secret from Secrets Manager (not SSM — never store secrets in Parameter Store)
SECRET_NAME=$(aws ssm get-parameter --name "/${PROJECT_NAME}/m2m/secret_name" --query "Parameter.Value" --output text)
CLIENT_SECRET=$(aws secretsmanager get-secret-value --secret-id "$SECRET_NAME" --query "SecretString" --output text)

cat > oauth_config_temp.json << EOF
{
  "customOauth2ProviderConfig": {
    "oauthDiscovery": { "discoveryUrl": "$ISSUER_URL" },
    "clientId":     "$CLIENT_ID",
    "clientSecret": "$CLIENT_SECRET"
  }
}
EOF

aws bedrock-agentcore-control create-oauth2-credential-provider \
    --name "$PROVIDER_NAME" \
    --credential-provider-vendor "CustomOauth2" \
    --oauth2-provider-config-input "file://oauth_config_temp.json"

rm -f oauth_config_temp.json
```

The `oauth_provider_name` is stored in SSM by Terraform. The agents read it at runtime to exchange tokens.

---

## Workload Identity

Workload identity lets an agent container obtain OAuth tokens for human users without handling passwords. Each agent gets its own named workload identity.

```python
# combined_terraform/Agentcore_identity/create_workload_identity.py

from bedrock_agentcore.services.identity import IdentityClient
import boto3, os

PROJECT_NAME    = os.environ['PROJECT_NAME']
AWS_REGION      = os.environ.get('AWS_REGION', 'us-east-1')
identity_client = IdentityClient(AWS_REGION)
ssm_client      = boto3.client('ssm', region_name=AWS_REGION)

for param_key in [
    f'/{PROJECT_NAME}/agentcore/trend-workload-name',
    f'/{PROJECT_NAME}/agentcore/anomaly-workload-name',
]:
    name = ssm_client.get_parameter(Name=param_key)['Parameter']['Value']

    try:
        wi = identity_client.create_workload_identity(name=name)
        print(f"✓ Created workload identity: {name}")
        print(f"  ARN: {wi['workloadIdentityArn']}")
    except Exception as e:
        if "already exists" in str(e).lower():
            print(f"✓ Workload identity already exists: {name}")
        else:
            raise
```

---

## Token Flow

At agent invocation time, the agent needs an OAuth token to call the MCP gateway. The full flow:

```
1. Agent is invoked by AgentCore runtime
2. Agent calls IdentityClient.get_workload_access_token(workload_name)
   → Returns a short-lived workload access token
3. Agent exchanges workload token for an M2M OAuth token
   via @requires_access_token(provider_name, scopes, auth_flow="M2M")
4. Agent creates MCP client with the M2M token as Bearer
5. Every tool call goes through: MCP client → gateway (JWT validated) → Lambda
```

```python
from bedrock_agentcore.services.identity import IdentityClient
from bedrock_agentcore.identity.auth import requires_access_token

def get_workload_token(workload_name: str) -> str:
    identity_client = IdentityClient(AWS_REGION)
    response = identity_client.get_workload_access_token(workload_name=workload_name)
    return response.get('workloadAccessToken')

async def fetch_m2m_token(provider_name: str, scopes: list[str]) -> str:
    @requires_access_token(provider_name=provider_name, scopes=scopes, auth_flow="M2M")
    async def _fetch(*, access_token: str):
        return access_token
    return await _fetch(access_token="")
```

---

## Adding a New Tool Lambda

1. Create the Lambda in `{agent_folder}/tools/{tool_name}.py`.
2. Define the tool Terraform resource in a `*_lambdas.tf` file.
3. Write a JSON schema in `{agent_folder}/schemas/{tool_name}.json`.
4. Add an SSM parameter storing the Lambda ARN (written by Terraform after creation).
5. Add an entry to `lambda_targets` in `gateway.py`.
6. Re-run `gateway.py` — the idempotent logic will detect the missing target and register it without touching existing targets.

---

[← 06: Bedrock Agents](06-bedrock-agents-strands.md) &nbsp;·&nbsp; [🏠 Home](README.md) &nbsp;·&nbsp; [Next: 08 — Athena Views →](08-athena-views-sql.md)
