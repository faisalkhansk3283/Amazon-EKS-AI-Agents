# Component Deep Dive: LiteLLM

**Role in this repo:** the single abstraction layer that lets the exact same agent code run against a self-hosted open-source model or Amazon Bedrock, with only a `model_id` string changing.

---

## What it actually is

LiteLLM is an OpenAI-API-compatible proxy. Your agent code sends requests in the standard OpenAI chat-completions format to **one URL**; LiteLLM looks at the `model` field and routes the request to whatever backend that name maps to — a self-hosted vLLM server, Amazon Bedrock, or (in principle) any other supported provider.

```
Agent pod (any track)
    OpenAIModel(base_url=LITELLM_BASE_URL)
                │
                ▼
┌─── LiteLLM Proxy (litellm namespace) ──────────┐
│  qwen2-5-3b-neuron → vLLM                       │
│  nova-lite         → Bedrock (Pod Identity)     │
│  claude-sonnet-4-5 → Bedrock (Pod Identity)     │
└─────────────────────────────────────────────────┘
```

## Why this is the load-bearing idea of the whole repo

Every "self-managed vs. integrated" comparison in this repo boils down to one config value. Compare the two `OpenAIModel` constructions:

```python
# Self-managed
model = OpenAIModel(
    client_args={"base_url": litellm_base_url, "api_key": "not-needed"},
    model_id="qwen2-5-3b-neuron",
    params={"max_tokens": 1024, "temperature": 0.3},
)

# Integrated
model = OpenAIModel(
    client_args={"base_url": os.environ["LITELLM_BASE_URL"], "api_key": "not-needed"},
    model_id=os.environ.get("MODEL_ID", "nova-lite"),
    params={"max_tokens": 1024, "temperature": 0.3},
)
```

Same SDK class (`OpenAIModel`), same client shape, one different string. No new authentication code, no new client library, no rewiring of the agent loop. This is the entire justification for putting LiteLLM in the architecture rather than pointing agents directly at vLLM or directly at `boto3`/Bedrock.

## Credential handling

Agent pods carry **no model-provider credentials at all**. `api_key` is set to a placeholder (`"not-needed"`) because LiteLLM itself holds the real credentials:

- For the vLLM route: no credentials needed (in-cluster service).
- For the Bedrock route: **Pod Identity** grants the LiteLLM pod an IAM role with Bedrock invoke permissions. The agent never sees an AWS credential.

This keeps the security boundary in one place — the proxy — rather than scattered across every agent pod that might need to call a model.

## Model-specific quirks handled at the proxy level, not the agent level

Example from this repo: Qwen3's extended "thinking" output is disabled centrally via `extra_body.chat_template_kwargs.enable_thinking = false` in the LiteLLM model config, rather than in agent code. Any agent using that `model_id` gets the behavior automatically — a good illustration of why centralizing model routing pays off: quirks get fixed once, not once per agent.

## Interaction with Langfuse

LiteLLM has its own Langfuse callback and forwards **proxy-level spans** into the same Langfuse project the agent traces to. This means every model call is visible twice, from two angles:

- The agent's OTel span (`invoke_agent Strands Agents` → LLM call): shows what the agent sent/received from the agent's perspective.
- LiteLLM's own trace (`litellm-acompletion`): shows the actual backend call and its raw latency.

Comparing the two lets you isolate whether latency comes from your orchestration code or from the model backend itself.

## Known version constraint (structured output / evaluation)

The LiteLLM version pinned in this workshop's Terraform returns Bedrock structured output (used for JSON-schema-based judge scoring in Langfuse evaluators) in a shape that Langfuse can parse **only for a specific set of models** — including `claude-sonnet-4-5`, but **not** `claude-sonnet-4-6`. Using 4.6 as a judge model with this LiteLLM version causes every evaluation to fail with `"No output generated."` To use 4.6, you'd need to upgrade LiteLLM to a version with native structured-output support for it and update the route in `terraform/litellm.tf`. Worth remembering any time you bump the judge model version.

## Deployment topology

- Namespace: `litellm`
- Service: `litellm.litellm:4000` — the one URL every agent, in both tracks, talks to
- Not deployed by the labs in this repo — provisioned once via Terraform as shared infrastructure, reused by every module

```bash
kubectl get pods -n litellm
```

## Common pitfalls

| Symptom | Likely cause |
|---|---|
| Agent gets connection refused | `LITELLM_BASE_URL` env var missing or pointing at the wrong namespace/port |
| Bedrock calls fail with an auth error | Pod Identity binding on the LiteLLM ServiceAccount is missing or scoped to the wrong IAM role |
| Switching `model_id` has no effect | Model alias not defined in LiteLLM's own config (`litellm.tf` / config map) — the proxy doesn't know that name |
| Judge/evaluation calls return "No output generated" | Judge model not in LiteLLM's structured-output-compatible list for this proxy version (see above) |

## Related docs

- [`langfuse.md`](langfuse.md) — how LiteLLM's proxy spans surface alongside agent traces
- [`agentcore.md`](agentcore.md) — the managed services LiteLLM does *not* front (Memory, Browser, Code Interpreter are called directly via `boto3`/Pod Identity, not through LiteLLM)