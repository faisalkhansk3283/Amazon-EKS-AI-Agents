# Component Deep Dive: MCP (Model Context Protocol)

**Role in this repo:** the tool-access layer for domain-specific business logic in both tracks (module 600 self-managed, reused unchanged in module 500 integrated).

---

## What it actually is

MCP is an open protocol for how agents **discover and call tools over the network**, instead of hardcoding tool functions directly into the agent's source file. A server advertises a set of tools (name, description, input schema); a client connects, asks "what tools do you have?", and gets back a live list it can hand straight to the agent's tool-calling mechanism.

This decouples tool logic from agent logic:
- Tools can be updated and redeployed **without touching the agent at all**.
- The same tool server can be reused across multiple agents.
- The agent doesn't need to know at build time what tools will exist at runtime.

## Server side: `FastMCP`

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("AnyCompany Tools")

@mcp.tool()
def lookup_order(order_id: str) -> dict:
    """Look up order status, tracking, and details by order ID."""
    ...

@mcp.tool()
def check_inventory(product_name: str) -> dict:
    """Check stock availability for a product."""
    ...
```

- `@mcp.tool()` works the same way as Strands' own `@tool` decorator: write a plain Python function, FastMCP auto-generates the tool schema from the function signature and docstring — no manual JSON schema authoring.
- `mcp.streamable_http_app()` returns an ASGI app that `uvicorn` serves directly — the MCP server is just a normal long-running HTTP service, deployed with a standard Kubernetes Deployment + Service, no special infrastructure.
- This module's server defines three tools (`lookup_order`, `check_inventory`, `initiate_return`), each backed by its own independent mock dataset.

## Client side: dynamic tool discovery

The agent doesn't hardcode which tools exist. At startup, it asks the MCP server:

```python
mcp_client = MCPClient(lambda: streamablehttp_client(mcp_server_url))
mcp_client.__enter__()

mcp_tools = mcp_client.list_tools_sync()
print(f"Discovered {len(mcp_tools)} MCP tools: {[t.tool_name for t in mcp_tools]}")

agent = Agent(model=model, system_prompt=SYSTEM_PROMPT,
              tools=[search_products, *mcp_tools])
```

- `list_tools_sync()` pulls the tool schema from the server at startup — the agent process genuinely does not know these tools exist until this call runs.
- The client connection is **kept open for the whole process lifetime**. This is a long-running HTTP service, not a per-request script, so a persistent connection avoids reconnect overhead on every chat message.
- Local tools (like `search_products`, a Milvus RAG tool) and remote MCP tools are passed into the same `tools=[...]` list — from the LLM's perspective, there's no difference between a locally-defined tool and a network-discovered one.

## What changes when MCP is introduced (module 600 vs. module 500)

Before MCP, order-lookup logic lived in a local `tools.py` file with hardcoded mock data. After MCP:
- `tools.py` is deleted from the agent's codebase entirely.
- All tool logic moves to a separate, independently deployable MCP server.
- The agent gains a clean separation of concerns: it owns *reasoning*, the MCP server owns *domain actions*.

## Rule of thumb: MCP vs. AgentCore sandboxes

This repo draws a clear line, worth internalizing:

- **Domain-specific business logic** (order lookup, inventory checks, initiating returns) → MCP, running on your own infrastructure (EKS). Nobody else can pre-build these tools for you; they're specific to your business.
- **Generic, potentially risky tools** (arbitrary Python execution, fetching arbitrary web pages) → managed sandboxes (AgentCore Code Interpreter / Browser, see [`agentcore.md`](agentcore.md)). These are the same for every team and dangerous to run next to your own production workloads.

Notice that MCP **doesn't change** between the self-managed and integrated tracks — it's explicitly called out as staying "unchanged" when the rest of the stack moves to Bedrock/AgentCore. Domain tools don't belong in a managed AI service; they belong wherever your business logic already lives.

## Deployment

```bash
cd ~/environment/modules/20-self-managed/600-agent-tools-mcp/mcp-server
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/mcp-server --timeout=60s
```

In the integrated multi-agent module, the MCP server is deployed into the `agents` namespace alongside the Order Agent, which connects to it at `mcp-server.agents.svc.cluster.local:8080/mcp`.

## Common pitfalls

| Symptom | Likely cause |
|---|---|
| Agent starts with "Discovered 0 MCP tools" | `mcp_server_url` unreachable — check the MCP Service/namespace/port match what the agent's env points at |
| Tool calls succeed but return stale/wrong mock data | MCP server pod wasn't redeployed after a tool logic change — tools live entirely server-side now |
| Agent hallucinates a tool that doesn't exist | `list_tools_sync()` failed silently at startup and the agent fell back to no tools — check startup logs, not just runtime behavior |
| Connection drops mid-conversation | Client connection lifetime assumption broken — verify `mcp_client.__enter__()` is called once at process start, not per-request |

## Related docs

- [`agentcore.md`](agentcore.md) — the managed-sandbox counterpart for generic/risky tools
- [`a2a.md`](a2a.md) — how MCP-backed specialist agents (e.g. an "Order Agent") get composed into a multi-agent system