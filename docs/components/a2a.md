# Component Deep Dive: A2A (Agent-to-Agent Protocol)

**Role in this repo:** multi-agent orchestration in both tracks (module 700 self-managed, module 500 integrated) — splitting one overloaded agent into focused specialists coordinated by an orchestrator.

---

## Why this exists: the problem with one big agent

A single agent handles simple workflows fine. As you add tools and responsibilities, two things degrade:

- **The system prompt grows**, and the LLM starts confusing similar tools or forgetting instructions buried deep in a long prompt.
- **Coupling increases** — updating one capability (say, changing the return policy) means touching the entire agent's prompt and risking regressions in unrelated behavior (order lookups, product search).

A2A's answer: give each agent **one job, one short prompt**, and add an orchestrator whose only responsibility is routing.

## Architecture used in this repo

```
Order Agent      — lookups and returns (via MCP)
Product Agent    — product/FAQ search (via Milvus)         [self-managed]
Sandbox Agent    — code execution + web fetch (via AgentCore)  [integrated]
Orchestrator     — routes queries to the right specialist over A2A
```

The orchestrator exposes a normal `/chat` HTTP endpoint that the UI calls directly. Specialists do **not** expose a chat endpoint — they speak A2A JSON-RPC and are only ever called by the orchestrator (or, in principle, other agents).

## The three components of `a2a-sdk`

### 1. `AgentExecutor.execute()` — the core loop

```python
class OrderAgentExecutor(AgentExecutor):
    def __init__(self):
        self.mcp_client = mcp_client
        self.mcp_client.__enter__()
        self.agent = Agent(
            model=model, system_prompt="You handle order inquiries...",
            tools=self.mcp_client.list_tools_sync(),
        )

    async def execute(self, context: RequestContext, event_queue: EventQueue) -> None:
        query = context.get_user_input()
        reply = str(self.agent(query))
        await event_queue.enqueue_event(new_agent_text_message(reply))
```

Every specialist is, underneath, just a normal Strands `Agent` — same class used everywhere else in this repo — wrapped in a thin adapter that speaks A2A's wire format. There is no separate "multi-agent SDK" to learn; A2A is a transport, not a new agent framework.

### 2. `AgentCard` — self-description for discovery

```python
agent_card = AgentCard(
    name="Order Agent",
    url="http://order-agent.default.svc.cluster.local:8081",
    capabilities=AgentCapabilities(streaming=False),
    default_input_modes=["text"], default_output_modes=["text"],
    skills=[AgentSkill(id="orders", name="Order Management", ..., tags=["orders"])],
    version="1.0.0", description="Handles order status lookups and return processing",
)
```

Published at `/.well-known/agent.json`. Any other agent or client can fetch this to learn what a specialist does and how to reach it — the same discoverability idea as MCP's tool listing, applied one level up (agents discovering agents, not agents discovering tools).

### 3. `A2AStarletteApplication` — the wire protocol

Wraps the `AgentExecutor` + `AgentCard` as a standard ASGI application speaking A2A JSON-RPC. Deployed exactly like any other HTTP service — a normal Kubernetes Deployment + Service, nothing A2A-specific about the infrastructure.

## What the orchestrator does differently

The orchestrator treats each specialist **as a tool**:

```
ask_order_agent(query)    → A2A call to Order Agent, returns its reply
ask_product_agent(query)  → A2A call to Product Agent, returns its reply
```

From the orchestrating LLM's perspective, calling a specialist agent looks exactly like calling any other `@tool`-decorated function — it just happens that the "tool" is itself a full agent running as a separate process. This is why the pattern composes: a specialist agent could itself orchestrate sub-specialists, though this repo keeps it to one level deep.

## Tracing implications

Because each specialist is a separate process, the A2A dispatch crosses a **process boundary**, and each specialist's work shows up as its **own separate top-level trace** in Langfuse — not nested inside the orchestrator's trace. What you *do* see nested inside the orchestrator's trace is the `ask_order_agent` / `ask_product_agent` tool-call span itself (the dispatch), while the specialist's internal reasoning (its own LLM calls, its own tool calls) lives in a sibling trace. When debugging a multi-agent flow, expect to look at two (or more) traces side by side, not one deep tree.

## What changes between self-managed and integrated A2A (module 700 → module 500)

| | Self-Managed | Integrated |
|---|---|---|
| Model | Qwen2.5-3B via LiteLLM | Nova via LiteLLM |
| Only agent-side diff | `model_id="qwen2-5-3b-neuron"` | `model_id="nova-lite"` |
| Order specialist | MCP on EKS | MCP on EKS (**unchanged**) |
| Second specialist | Product Agent (Milvus RAG) | Sandbox Agent (Code Interpreter + Browser) |
| Orchestrator memory | None | AgentCore Memory (per-customer session) |
| Namespace | `default` | `agents` |

Notice again: MCP-backed specialists don't change. Only the specialists that talk to a *model* or a *managed capability* change their backend.

## Deployment

```bash
cd ~/environment/modules/20-self-managed/700-multi-agent-a2a/a2a-agents
envsubst < k8s-specialists.yaml  | kubectl apply -f -
envsubst < k8s-orchestrator.yaml | kubectl apply -f -

kubectl rollout status deployment/order-agent        --timeout=120s
kubectl rollout status deployment/product-agent      --timeout=120s
kubectl rollout status deployment/orchestrator-agent --timeout=120s
```

## Common pitfalls

| Symptom | Likely cause |
|---|---|
| Orchestrator can't reach a specialist | `AgentCard.url` doesn't match the specialist's actual in-cluster Service DNS name/port |
| Orchestrator picks the wrong specialist | `AgentSkill` description too vague or overlapping between specialists — same "docstring is the selection heuristic" issue as MCP tools |
| Specialist trace never appears in Langfuse | Specialist's own Langfuse client not wired (`get_client()`/`flush()`) — each process needs its own instrumentation, it doesn't inherit the orchestrator's |
| Context from turn 1 lost when routing to a different specialist in turn 2 | No shared memory across specialists (self-managed) — the integrated track solves this via AgentCore Memory scoped to the orchestrator |

## Related docs

- [`mcp.md`](mcp.md) — how the Order specialist accesses its tools
- [`agentcore.md`](agentcore.md) — Sandbox Agent's backend in the integrated track, and orchestrator-level session memory
- [`langfuse.md`](langfuse.md) — why multi-agent traces appear as multiple linked traces, not one tree