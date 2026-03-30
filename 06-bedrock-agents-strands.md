# 06 — Bedrock Agents and the Strands SDK

> **Quick reference:** [Strands imports](#strands-imports) · [@tool decorator](#tool-decorator) · [BedrockModel](#bedrockmodel) · [Agent constructor](#agent-constructor) · [BedrockAgentCoreApp](#bedrockagentcoreapp) · [MCP client](#mcp-client) · [SSM with lru_cache](#ssm-with-lru_cache) · [.bedrock_agentcore.yaml](#bedrockagentcoreyaml)

---

## Overview

Agents are long-running Python containers deployed on Amazon Bedrock AgentCore. They receive a payload, run a multi-step reasoning loop via the Strands SDK, call Lambda tools through the MCP gateway, and return a structured result.

Two agents are deployed in this example project:

| Agent | Entry point | Purpose |
|-------|------------|--------|
| Trend Analysis Agent | `agents/trend-analysis-agent/agent/trend_analysis_agent.py` | Identifies sales trends, growth drivers, and period-over-period changes |
| Anomaly Detection Agent | `agents/anomaly-detection-agent/agent/anomaly_detection_agent.py` | Detects unusual spikes or drops in sales, revenue, or discounts |

---

## Strands Imports

```python
from strands import Agent, tool
from strands.models import BedrockModel
from strands.tools.mcp.mcp_client import MCPClient
from mcp.client.streamable_http import streamablehttp_client
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.identity.auth import requires_access_token
from bedrock_agentcore.services.identity import IdentityClient
from bedrock_agentcore.runtime import BedrockAgentCoreContext
```

Install these in `requirements.txt` inside the agent folder:

```
strands-agents
strands-agents-tools
bedrock-agentcore
bedrock-agentcore-starter-toolkit
mcp
boto3
```

---

## `@tool` Decorator

Mark any Python function as a tool the agent can call by adding `@tool`. The function's docstring becomes the tool description passed to the model.

```python
from strands import tool

@tool
def get_monthly_sales(product_id: str, year: int, month: int) -> dict:
    """
    Fetch monthly sales totals for a specific product.
    Returns gross sales, quantity sold, and discount amounts for the month.

    Args:
        product_id: The product's unique identifier
        year: Year (2020–2030)
        month: Month number (1–12)
    """
    query = f"""
        SELECT
            SUM(o.gross_sales)                          AS gross_sales,
            SUM(o.quantity)                             AS units_sold,
            COALESCE(SUM(d.discount_amount), 0.0)       AS total_discounts,
            SUM(o.gross_sales) - COALESCE(SUM(d.discount_amount), 0.0) AS net_sales
        FROM orders o
        LEFT JOIN order_discounts d ON o.order_id = d.order_id
        WHERE o.product_id = '{product_id}'
          AND YEAR(o.order_date)  = {year}
          AND MONTH(o.order_date) = {month}
    """
    rows = execute_athena_query(query)
    return {"rows": rows, "count": len(rows)}
```

**Rules for `@tool` functions:**
- Accept and return only JSON-serializable types.
- Write a clear docstring — this is what the LLM reads to decide when to call the tool.
- Keep each tool focused on one operation.
- Never call other `@tool` functions from within a tool — let the agent orchestrate sequencing.

---

## BedrockModel

Configure the underlying LLM. Use separate models for orchestration (where reasoning matters) vs. fast inline tasks.

```python
from strands.models import BedrockModel
from botocore.config import Config as BotocoreConfig
import boto3

ORCHESTRATION_MODEL_ID = os.environ.get(
    "ORCHESTRATION_MODEL_ID",
    "us.anthropic.claude-sonnet-4-20250514-v1:0"
)

bedrock_client = boto3.client(
    "bedrock-runtime",
    region_name=AWS_REGION,
    config=BotocoreConfig(
        retries={"max_attempts": 3},
    ),
)

model = BedrockModel(
    model_id=ORCHESTRATION_MODEL_ID,
    boto3_client=bedrock_client,
)
```

---

## Agent Constructor

```python
from strands import Agent

SYSTEM_PROMPT = """
You are a retail sales trend analysis agent. Your task is to identify what is
driving growth or decline in sales for a specific product and region.

Execute the analysis in exactly 4 steps:
1. Fetch monthly sales data for the product and requested period.
2. Compare actuals to the previous period to calculate period-over-period change.
3. Break down the change by sales channel (online, wholesale, retail).
4. Rank the contributing channels by their impact on the observed trend.

Do not skip steps. Do not generate data — always call the provided tools.
"""

agent = Agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=get_tools_list(mcp_client),   # list of @tool functions + MCP tools
)

# Invoke the agent
result = agent("Analyse sales trends for Product P-1042 in the North-East region for Q3 2025")
```

---

## BedrockAgentCoreApp

`BedrockAgentCoreApp` is the container entry point. It wires the `@app.entrypoint` function to the AgentCore runtime.

```python
from bedrock_agentcore import BedrockAgentCoreApp
from typing import Dict, Any

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload: Dict[str, Any]) -> Dict[str, Any]:
    """
    Called by AgentCore runtime on each invocation.
    payload contains the user request + any context fields.
    """
    # Unwrap nested payload that AgentCore may add
    if "input" in payload:
        payload = payload["input"]

    # Validate required fields
    product_id = payload.get("product_id")
    if not product_id:
        return {"error": "Missing required field: product_id"}

    # Run the agent
    result = agent(f"Analyse trends for product {product_id} in {payload.get('region')} for period {payload.get('period')}")
    return {"result": str(result)}
```

The module-level `app = BedrockAgentCoreApp()` is mandatory. The `@app.entrypoint` decorator registers the function as the invocation handler.

---

## SSM with `lru_cache`

Agents read SSM parameters repeatedly across invocations (OAuth URLs, gateway URL, workload name). Cache the results so SSM is called once per container lifetime:

```python
from functools import lru_cache
import boto3

@lru_cache(maxsize=32)
def get_parameter_value(param_name: str) -> str:
    """Fetch and cache an SSM parameter. Called once per cold start."""
    ssm = boto3.client('ssm', region_name=AWS_REGION)
    response = ssm.get_parameter(Name=param_name, WithDecryption=True)
    return response['Parameter']['Value']
```

Usage — all config via SSM parameter names from env vars:

```python
SSM_GATEWAY_URL_PARAM = os.environ.get('SSM_GATEWAY_URL_PARAM')

def get_gateway_url() -> str:
    return get_parameter_value(SSM_GATEWAY_URL_PARAM)```

---

## MCP Client

The MCP client connects the agent to the Lambda tools registered on the MCP gateway. Each tool call goes through HTTP to the gateway endpoint.

```python
from strands.tools.mcp.mcp_client import MCPClient
from mcp.client.streamable_http import streamablehttp_client

def create_mcp_client(gateway_url: str, access_token: str) -> MCPClient:
    """Create an MCP client connected to the gateway."""
    transport = streamablehttp_client(
        gateway_url,
        headers={"Authorization": f"Bearer {access_token}"},
    )
    return MCPClient(lambda: transport)

# Combine local @tool functions with remote MCP tools
def get_tools_list(mcp_client: MCPClient) -> list:
    local_tools = [get_monthly_sales, get_period_over_period_change]
    mcp_tools   = mcp_client.list_tools_sync()
    return local_tools + mcp_tools
```

The agent sees both local `@tool` functions and remote MCP tools the same way — as callable tools with names and descriptions. The agent decides which to call based on the task.

---

## Module-Level Invocation Context

`@tool` functions run inside the agent loop and do not receive the original invocation payload. Use a module-level dict to share context between `invoke()` and `@tool` functions:

```python
# Set once per invocation at module level
_INVOCATION_CONTEXT: Dict[str, Any] = {}

@app.entrypoint
def invoke(payload: Dict[str, Any]) -> Dict[str, Any]:
    global _INVOCATION_CONTEXT
    _INVOCATION_CONTEXT = {
        "product_id": payload["product_id"],
        "region":     payload["region"],
        "period":     payload["period"],
    }
    result = agent(f"Analyse {_INVOCATION_CONTEXT['product_id']}...")
    return result

@tool
def get_current_context() -> dict:
    """Return the current analysis context (product_id, region, period)."""
    return _INVOCATION_CONTEXT
```

---

## System Prompt Best Practices

1. **Be explicit about steps.** Number the steps. Agents follow numbered instructions far more reliably than prose.
2. **Ban hallucination explicitly.** Add: "Do not generate data. Always call the provided tools to retrieve information."
3. **Define the output format.** Specify the exact JSON shape to return — agents are much more consistent when the schema is in the system prompt vs. inferred.
4. **Keep it concise.** Every token in the system prompt is context overhead on every call.

---

## `.bedrock_agentcore.yaml`

Each agent folder requires a `.bedrock_agentcore.yaml` for `agentcore configure` to build and push the container.

```yaml
# agents/trend-analysis-agent/agent/.bedrock_agentcore.yaml

agentName: retail-analytics-dev-trend-analysis-agent
entrypoint: trend_analysis_agent.py   # Python file with BedrockAgentCoreApp
platform: linux/arm64                  # arm64 is ~20% cheaper than x86_64

ecr:
  repository: retail-analytics-dev-trend-analysis-agent  # ECR repo created by agentcore configure
  region: us-east-1

executionRole: arn:aws:iam::123456789:role/retail-analytics-dev-bedrock-agentcore-role

authorizerConfiguration:
  customJWTAuthorizer:
    discoveryUrl: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123/.well-known/openid-configuration
    allowedClients:
      - abc123clientid
      - xyz789clientid
```

This file is written by `agentcore configure` and overwritten on each re-configure. Do not hand-edit — re-run `agentcore configure` with updated args instead.

---

## Agent Structure Recap

```
{agent_name}/
├── agent/
│   ├── trend_analysis_agent.py     # BedrockAgentCoreApp + @tool functions
│   ├── .bedrock_agentcore.yaml     # Container/deploy config
│   ├── requirements.txt            # strands, bedrock-agentcore, boto3
│   └── Dockerfile                  # (optional — agentcore generates one)
├── tools/
│   └── {tool_name}.py              # Standalone tool Lambda functions
└── schemas/
    └── {tool_name}.json            # JSON schema for each tool (used by gateway)
```

---

[← 05: Async Lambda Architecture](05-async-lambda-architecture.md) &nbsp;·&nbsp; [🏠 Home](README.md) &nbsp;·&nbsp; [Next: 07 — AgentCore Gateway & Identity →](07-agentcore-gateway-identity.md)
