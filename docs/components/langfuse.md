# Component Deep Dive: Langfuse

**Role in this repo:** observability backend for every agent module in `self-managed/` and `integrated/`. Used unmodified across both tracks — the whole point is that the same tracing setup works regardless of which model backend is in play.

---

## What it actually is

Langfuse is an open-source LLM observability platform. It ingests **OpenTelemetry (OTel) spans** and renders them as **traces**: a hierarchical, timestamped view of everything that happened during one agent invocation — LLM calls, tool calls, retries, latencies, token counts, and costs.

Think of it as Jaeger/Datadog APM, but with first-class understanding of LLM-specific concepts: prompt/completion pairs, tool-call boundaries, token usage, and cost attribution.

## Why it matters for agents specifically

A plain application log tells you an agent responded. It does not tell you:

- How many LLM calls it took to answer one question
- Whether the agent called the right tool, or hallucinated one that doesn't exist
- Where the latency actually is — model inference, tool execution, or network
- What was literally sent to and received from the LLM at each step

Without this, "the agent gave a wrong answer" is undebuggable. With it, you can point at the exact span where things went sideways.

## How it integrates with Strands

With the `strands-agents[otel]` extra installed, the Strands SDK **automatically emits OTel spans** for the agent loop, every LLM call, and every tool execution — no manual instrumentation required. Langfuse ingests these via its OTel-compatible endpoint.

Minimal wiring pattern used in every module (`agent.py`):

```python
from langfuse import get_client

langfuse = get_client()
if langfuse.auth_check():
    print("Langfuse connected successfully")
else:
    print("WARNING: Langfuse authentication failed - traces will not be captured")

# ... agent construction and request handling ...

langfuse.flush()  # called at the end of every /chat request
```

- `get_client()` reads `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` / `LANGFUSE_BASE_URL` from environment (sourced from the `agent-config` ConfigMap).
- `auth_check()` converts a silent OTel auth failure into a loud, visible one at startup — worth keeping even outside a workshop context.
- `flush()` must be called before the HTTP response returns, or pending spans may be lost when the process later exits or the request handler returns.

## What a trace looks like

```
Trace: "Where is my order ORD-12345?"
├── Agent Loop
│   ├── LLM Call (qwen2-5-3b-neuron) — tool call decision
│   ├── Tool: lookup_order — ~2ms
│   └── LLM Call (qwen2-5-3b-neuron) — final response
└── Total: ~2s, 2 LLM calls, 1 tool call
```

In the integrated track, the same shape appears with `model_id=nova-lite` instead — the trace *structure* is identical; only the model label and latency profile change. This is a deliberate demonstration: swapping the backend doesn't change how you debug the agent.

## LiteLLM's own spans show up too

LiteLLM has its own Langfuse callback and logs every model call as a **separate top-level trace** named `litellm-acompletion`, distinct from the agent's `invoke_agent Strands Agents` trace. This lets you separate **agent-side latency** (reasoning, tool dispatch) from **proxy-side latency** (the actual model call), which is useful when tuning end-to-end response time. It also means: when filtering traces (e.g. for evaluation), you must filter on `invoke_agent Strands Agents`, not `litellm-acompletion` — the latter is not a conversation, just a proxy call log.

## Deployment specifics (self-managed track)

```bash
kubectl get pods -n langfuse   # web pod may show 1-2 restarts during first-boot DB migration — normal

echo "Langfuse UI: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
```

| Field | Value |
|---|---|
| Email | `admin@workshop.local` |
| Password | `workshop2025` |
| Project | AnyCompany Shop (pre-provisioned) |
| API keys | `pk-lf-workshop` / `sk-lf-workshop` (pre-provisioned) |

## Langfuse as an evaluation engine (module 750)

Langfuse doubles as the **LLM-as-a-Judge** harness in the self-managed track (see [`docs/components/evaluation.md`](evaluation.md) for the full write-up). In short: it can call out to a judge model through the same LiteLLM proxy the agent uses, score every trace on custom or built-in rubrics, and attach the score back onto the trace.

Two operational gotchas worth calling out here specifically:

1. **SSRF protection blocks in-cluster hosts by default.** Langfuse refuses to connect to RFC1918 (cluster-internal) addresses unless explicitly allowlisted — this applies to both `langfuse-web` and `langfuse-worker` pods:
   ```bash
   kubectl set env deployment/langfuse-web deployment/langfuse-worker -n langfuse \
     LANGFUSE_LLM_CONNECTION_WHITELISTED_HOST=litellm.litellm.svc.cluster.local
   ```
2. **Evaluators default to `Observations` scope, not `Traces`.** Recent Langfuse versions score individual spans by default. For whole-conversation scoring (the pattern used throughout this repo), the evaluator's **Target** must be explicitly set to `Traces`.

## Common pitfalls

| Symptom | Likely cause |
|---|---|
| No traces appear at all | `auth_check()` failed silently — check `LANGFUSE_PUBLIC_KEY`/`SECRET_KEY` env vars are populated |
| Traces appear but scores never populate | Evaluator filter doesn't match `invoke_agent Strands Agents` exactly, or `flush()` isn't being called before the judge call fires |
| "Blocked IP address detected" on evaluator runs | LiteLLM hostname not in `LANGFUSE_LLM_CONNECTION_WHITELISTED_HOST` on **both** langfuse-web and langfuse-worker |
| Accuracy scores near zero despite correct answers | Trace lacks a tool-call span, so the judge cannot verify grounding and (correctly) treats the claim as unverifiable |

## Related docs

- [`evaluation.md`](evaluation.md) — Langfuse-based and AgentCore-based judge scoring, side by side
- [`litellm.md`](litellm.md) — the proxy layer whose spans also land in Langfuse
- [`../COMMANDS.md`](../COMMANDS.md) — exact deploy/verify commands per module