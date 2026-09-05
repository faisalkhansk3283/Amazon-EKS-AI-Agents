# Component Deep Dive: Evaluation (LLM-as-a-Judge)

**Role in this repo:** automated quality scoring of agent conversations, implemented two ways — Langfuse-based (module 750, self-managed) and AgentCore Evaluations (module 550, integrated). This doc covers both and compares them directly, since they solve the identical problem with different tooling.

---

## The problem both approaches solve

A trace tells you *what* the agent did. It does not tell you *whether the answer was any good*. Reading conversations by hand doesn't scale past a few dozen. **LLM-as-a-Judge** closes the gap: a capable model grades another model's output against criteria you write, producing a numeric score plus reasoning, attached back onto the trace.

In both labs, the judge is `claude-sonnet-4-5` — a stronger, managed model auditing a cheaper one (`qwen2-5-3b-neuron` self-hosted, or `nova-lite` on Bedrock). That asymmetry is the point: a cheap model does the work, an expensive model checks it, and the agent code itself never changes.

---

## Approach 1: Langfuse (self-managed, module 750)

### How it's wired

Langfuse reads traces it already has (from the Observability module — nothing new gets instrumented) and calls out to the judge model **through the same LiteLLM proxy** the agent itself uses.

```
Customer-agent pod                 Langfuse (langfuse namespace)
  Strands → OTel spans  ───────▶   Trace (input / output)
                                          │
                                          ▼
                                   LLM-as-a-Judge evaluators
                                    cs-accuracy   (custom)
                                    Helpfulness   (managed)
                                    cs-safety     (custom)
                                          │ OpenAI API (judge model)
                                          ▼
                                   LiteLLM → Bedrock (claude-sonnet-4-5)
```

### Two evaluator types

- **Custom evaluators** (`cs-accuracy`, `cs-safety`): you write the full prompt, criteria, and scoring instructions yourself. Necessary because generic evaluators don't know this is a tool-using retail agent.
- **Managed evaluator** (`Helpfulness`): Langfuse ships this rubric pre-built — no prompt to write, just map the variables.

### Critical setup gotchas

1. **SSRF protection blocks the judge call by default.** Langfuse refuses connections to cluster-internal (RFC1918) addresses unless explicitly allowlisted, on **both** `langfuse-web` and `langfuse-worker`:
   ```bash
   kubectl set env deployment/langfuse-web deployment/langfuse-worker -n langfuse \
     LANGFUSE_LLM_CONNECTION_WHITELISTED_HOST=litellm.litellm.svc.cluster.local
   ```
2. **Evaluator target defaults to `Observations`, not `Traces`.** This lab scores whole conversations, so every evaluator's Target must be explicitly set to `Traces` — leaving the default silently scores individual LLM spans instead, and your filter/variable mapping won't match what you expect.
3. **Filter on the right trace name.** `invoke_agent Strands Agents` is the conversation trace (Strands names the agent's root span this way). `litellm-acompletion` is the proxy's own per-call trace and is **not** a conversation — filtering on it yields zero meaningful scores.
4. **Judge model version is pinned for a structural reason, not preference.** Langfuse asks the judge for a score via OpenAI's `response_format` JSON-schema mechanism. The pinned LiteLLM version returns Bedrock structured output in the shape Langfuse expects only for specific models, including `claude-sonnet-4-5` but **not** `claude-sonnet-4-6`. Using 4.6 fails every evaluation with `"No output generated."`

### Variable mapping

```
{{input}}  → Object Trace, Object Field Input
{{output}} → Object Trace, Object Field Output
```

Use the Evaluation Prompt Preview before saving — if both `{{input}}` and `{{output}}` show the same text (the customer's query), `{{output}}` is still mis-mapped.

### The finding, not a bug: accuracy scores low here

This agent's traces record the query and the final answer, but **not the `lookup_order` tool-call span**. The judge therefore cannot verify whether concrete order details were actually looked up or invented — and correctly flags possible fabrication, scoring accuracy low even when the agent's answer was factually correct. This is the single most important lesson from either evaluation lab: **a judge can only grade what your instrumentation captured.**

---

## Approach 2: AgentCore Evaluations (integrated, module 550)

### How it's wired

Instead of Langfuse, traces flow to **AgentCore Observability** (CloudWatch + X-Ray), and a managed service reads them directly:

```
Customer-agent pod (agents namespace)
  Strands → OTel spans
     │
     ├── LangfuseSpanProcessor ─────────▶ Langfuse (unchanged view, still works)
     │
     └── OTLPAwsSpanExporter (SigV4) ───▶ X-Ray OTLP endpoint
                                            │  (Transaction Search → aws/spans log group)
                                            ▼
                                          AgentCore Observability
                                            │
                                            ▼
                                          AgentCore Evaluations
                                            Builtin.Correctness
                                            Builtin.Helpfulness
                                            cs_accuracy (custom, TRACE level)
```

Notice this module **dual-exports** every span — to both X-Ray (for AgentCore Evaluations) and Langfuse (so the trace view you're used to keeps working). One `TracerProvider`, two exporters attached to it.

### Why not just use ADOT auto-instrumentation

AWS's standard docs suggest running under `opentelemetry-instrument`, but that installs ADOT's own global `TracerProvider`, which conflicts with Langfuse's own provider ownership. The fix used here: build one `TracerProvider` explicitly, register the AWS SigV4 X-Ray exporter on it, then hand that *same* provider to the Langfuse v4 client via `Langfuse(tracer_provider=provider)`, which attaches its own processor internally rather than creating a competing one.

### What AgentCore gives you over rolling your own judge

- **16 built-in evaluators** with fixed prompts and a fixed judge model — scores stay comparable across runs and across teams, unlike hand-written prompts that drift.
- **Three granularities**: span (one tool call), trace (one turn), session (whole conversation).
- **Two modes**: on-demand (spot checks, used in this lab) or online evaluation (continuous sampling of live traffic).
- Reads OTel traces from Strands/LangGraph **with no evaluation-specific code in the agent** beyond the dual-export wiring.

### Creating a custom evaluator (CLI, not a UI)

```bash
cat > /tmp/cs_accuracy_config.json <<'JSON'
{
  "llmAsAJudge": {
    "instructions": "You are a senior customer-service QA auditor... Context: {context} ... Agent response: {assistant_turn} ...",
    "ratingScale": {
      "numerical": [
        {"label": "Grounded", "definition": "Fully grounded in tool results, no invented details", "value": 1.0},
        {"label": "Partial",  "definition": "Mostly grounded, minor unsupported detail", "value": 0.5},
        {"label": "Invented", "definition": "Invented order details or ignored tool results", "value": 0.0}
      ]
    },
    "modelConfig": {"bedrockEvaluatorModelConfig": {"modelId": "us.anthropic.claude-sonnet-4-5-20250929-v1:0"}}
  }
}
JSON

aws bedrock-agentcore-control create-evaluator \
  --evaluator-name cs_accuracy --level TRACE \
  --evaluator-config file:///tmp/cs_accuracy_config.json \
  --query 'evaluatorId' --output text
```

Four hard constraints on the config, worth knowing before you write your own:
1. `--evaluator-name` must match `[a-zA-Z][a-zA-Z0-9_]{0,47}` — no hyphens (hence `cs_accuracy`, not `cs-accuracy`).
2. TRACE-level `instructions` **must** contain a placeholder (`{context}`, `{assistant_turn}`, `{tool_turn}`, etc.).
3. Placeholders use single braces, not double (unlike Langfuse's `{{input}}`/`{{output}}`).
4. A `modelConfig` judge model is required for custom evaluators — built-ins have one baked in.

### Running an on-demand evaluation

```bash
for E in "Builtin.Correctness" "Builtin.Helpfulness" "$CUSTOM_ID"; do
  aws bedrock-agentcore evaluate \
    --evaluator-id "$E" \
    --evaluation-input  file:///tmp/eval_input.json \
    --evaluation-target "{\"traceIds\":[\"$TRACE_ID\"]}" \
    --query 'evaluationResults[0].{evaluator:evaluatorName,score:value,label:label,reason:explanation}' \
    --output json
done
```

Each call is synchronous — scores return directly in CLI output and are **not persisted anywhere automatically**. That's intentional: on-demand is for spot checks. For continuously accumulating scores in the CloudWatch console, use `create-online-evaluation-config` instead (not covered in the base lab).

### Why accuracy scores well here, unlike the self-managed lab

This agent's dual-exported traces **do** include the `lookup_order` tool-call span, tagged with `session.id`. The judge can directly verify grounding — same judge, same methodology as the Langfuse lab, but richer traces produce a meaningfully different (and more trustworthy) accuracy score. This is the payoff of the lesson from the self-managed lab: instrument tool calls, not just the final answer.

---

## Side-by-side comparison

| | Self-managed (Langfuse) | Integrated (AgentCore) |
|---|---|---|
| Judge | `claude-sonnet-4-5` via LiteLLM, inside Langfuse | AgentCore Evaluations (managed) |
| Trace source | Langfuse | AgentCore Observability (dual-exported) |
| Evaluators | Custom + managed (Langfuse's library) | Built-in (16) + custom |
| Granularity | Trace only | Span, trace, or session |
| Where it runs | Langfuse UI (evaluator wizard) | `bedrock-agentcore` API / CLI |
| Persistence | Scores attach to the trace automatically | On-demand scores are ephemeral (CLI output only) unless you also configure online evaluation |
| Setup friction | SSRF allowlist, evaluator target default, judge model version pin | Resource-attribute wiring (`aws.service.type`, `cloud.resource_id`) so AgentCore can find the spans |

## The one lesson that applies to both

**Evaluation quality is capped by instrumentation quality.** If a tool call isn't traced, no judge — managed or self-hosted — can verify grounding for it. Before investing in evaluator prompt engineering, verify your tool-call spans are actually present in whichever trace store you're scoring against.

## Related docs

- [`langfuse.md`](langfuse.md) — the trace store the self-managed evaluators read
- [`agentcore.md`](agentcore.md) — Memory and Tools, the other two AgentCore pieces used in this repo
- [`../COMMANDS.md`](../COMMANDS.md) — full copy-paste command sequences for both evaluation labs