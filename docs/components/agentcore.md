# Component Deep Dive: Amazon Bedrock AgentCore

**Role in this repo:** the integrated track's managed replacement for self-hosted memory, sandboxed tool execution, and evaluation — everything that's generic enough for AWS to run for you, as opposed to domain-specific logic (which stays on MCP).

AgentCore is not one service — it's a suite. This repo uses three parts of it: **Memory**, **Browser + Code Interpreter** (sandboxed tools), and **Evaluations**. Each is covered separately below.

---

## AgentCore Memory (module 300, integrated)

**Problem it solves:** the same one Milvus solves in module 500 self-managed — recalling earlier turns in a session — but as a managed service via `boto3`, with no vector database to run yourself.

### Why this isn't a 1:1 swap for Milvus memory

AgentCore Memory and Milvus solve overlapping but not identical problems:

| | Self-managed: Milvus | Integrated: AgentCore Memory |
|---|---|---|
| What it stores | Conversation events per customer session | Conversation events per customer session |
| Access pattern | Vector search *or* scalar query (your choice) | Event history, retrieval by session/actor (fixed API shape) |
| Use case | Session memory *and* can extend to RAG | Session memory only — RAG still needs Milvus or Bedrock Knowledge Bases |

AgentCore Memory is purpose-built for conversational session memory specifically. It does not replace a vector database for product-catalog RAG — that job still needs Milvus or Bedrock Knowledge Bases even in the integrated track.

### API shape

```python
def record_turn(actor_id, session_id, user_message, assistant_message):
    """Persist one turn (user + assistant) as an event."""
    _client.create_event(
        memoryId=MEMORY_ID, actorId=actor_id, sessionId=session_id,
        eventTimestamp=datetime.now(timezone.utc),
        payload=[
            {"conversational": {"role": "USER", "content": {"text": user_message}}},
            {"conversational": {"role": "ASSISTANT", "content": {"text": assistant_message}}},
        ],
    )

def recent_turns(actor_id, session_id, max_results=20):
    """Return recent turns, oldest first."""
    resp = _client.list_events(memoryId=MEMORY_ID, actorId=actor_id, sessionId=session_id,
                                maxResults=max_results, includePayloads=True)
    # list_events returns newest-first; reverse to read chronologically
```

- `actorId` = the customer's identity (email, customer ID). `sessionId` = one conversation.
- Events **expire after 30 days** by default (configured in `terraform/agentcore.tf`) — this is a TTL-based store, not permanent storage.
- Wrapping each call in an `@observe` decorator is what makes these calls show up as spans in Langfuse — Strands' own OTel instrumentation only covers the agent loop itself, not arbitrary `boto3` calls you add. Without `@observe`, memory reads/writes would be invisible in traces.
- No new SDK: this is plain `boto3` against the `bedrock-agentcore` service — no special client library beyond what you'd already have for any other AWS service call.

### Inspecting stored events directly

```bash
aws bedrock-agentcore list-events \
  --memory-id "$MEMORY_ID" --actor-id "workshop-user" --session-id "$SESSION_ID" \
  --include-payloads
```

---

## AgentCore Browser + Code Interpreter (module 400, integrated)

**Problem it solves:** giving the agent access to arbitrary code execution or live web page fetching, without running that risk inside your own cluster next to production workloads.

### Why these are managed, not self-built

Both fit the same conceptual slot as MCP tools, but the risk profile is different: arbitrary Python execution or fetching attacker-controlled URLs is dangerous to run on infrastructure you also use for anything else. AgentCore provides isolated, ephemeral sandboxes specifically for this class of tool.

### Shape of both tools: start → invoke → stop

```python
@tool
@observe(name="sandbox.run_python")
def run_python(code: str) -> str:
    """Execute Python code in a sandboxed AgentCore code interpreter and return stdout.
    Use this for calculations, data processing, or generating small reports.
    The sandbox has no access to internal AWS services.
    """
    session = _client.start_code_interpreter_session(
        codeInterpreterIdentifier=CODE_INTERPRETER_ID, sessionTimeoutSeconds=300)
    session_id = session["sessionId"]
    try:
        resp = _client.invoke_code_interpreter(
            codeInterpreterIdentifier=CODE_INTERPRETER_ID, sessionId=session_id,
            name="executeCode", arguments={"language": "python", "code": code})
        # ... read back stdout chunks ...
    finally:
        _client.stop_code_interpreter_session(
            codeInterpreterIdentifier=CODE_INTERPRETER_ID, sessionId=session_id)
```

- Three-step lifecycle every time: `start_*_session` → `invoke` → `stop_*_session` in a `finally` block, so sessions are always cleaned up even on error.
- **One session per tool call** in this lab, for clarity. A production deployment would cache sessions per agent or per conversation to avoid paying the session-startup cost on every single call — the workshop trades a bit of latency for simplicity.
- `fetch_webpage` follows the same shape using `BrowserClient` (from `bedrock-agentcore-starter-toolkit`), which handles the Chrome DevTools Protocol WebSocket connection for you.
- Browser sessions take noticeably longer to start than Code Interpreter sessions (a fresh isolated browser has more to spin up than a Python sandbox) — that latency is, functionally, the price of isolation.

### Why `@observe` matters here specifically

Without the `@observe` decorator, Langfuse shows only that `run_python` or `fetch_webpage` was called — not what code was executed or what came back. With it, the trace's Input shows the exact generated code (or requested URL) and the Output shows the actual stdout (or scraped page text). This level of detail is what lets you catch cases like a webpage returning a JavaScript-rendered empty shell instead of real content — invisible without inspecting the span payload directly.

### Deployment

```bash
kubectl get configmap agent-tools -n agents -o yaml   # confirm AGENTCORE_BROWSER_ID, AGENTCORE_CODE_INTERPRETER_ID
```

The agent's ServiceAccount already has `StartBrowserSession` / `StartCodeInterpreterSession` scoped to these specific resource ARNs — least-privilege, not a broad AgentCore grant.

---

## AgentCore Evaluations (module 550, integrated)

Covered in full detail in [`evaluation.md`](evaluation.md), including the self-managed vs. integrated comparison. Summary: this is AWS's managed LLM-as-a-Judge service, reading OpenTelemetry spans through AgentCore Observability (CloudWatch/X-Ray) instead of Langfuse, with 16 built-in evaluators plus support for custom ones.

---

## Common pitfalls (Memory + Tools)

| Symptom | Likely cause |
|---|---|
| `recent_turns` returns empty on turn 2 | `actorId`/`sessionId` mismatch between the write in turn 1 and the read in turn 2 |
| Memory calls invisible in Langfuse | Missing `@observe` decorator — Strands' auto-instrumentation doesn't cover custom `boto3` calls |
| `run_python`/`fetch_webpage` span shows no code/output detail | Same — `@observe` must wrap the tool function, stacked under `@tool` |
| Sandbox calls fail with an access-denied error | Agent ServiceAccount's IAM permissions don't include `Start*Session` for the specific `codeInterpreterIdentifier`/`browserIdentifier` in use |
| High latency on every single sandbox call | Expected for Browser (fresh isolated browser per session) — this is inherent to the isolation model, not a bug; cache sessions in production |

## Related docs

- [`evaluation.md`](evaluation.md) — full AgentCore Evaluations deep dive and comparison with Langfuse-based judging
- [`mcp.md`](mcp.md) — the domain-tool counterpart that stays on your own infra instead of moving to AgentCore
- [`litellm.md`](litellm.md) — note that AgentCore services are called directly via `boto3`/Pod Identity, **not** routed through LiteLLM (LiteLLM only fronts model inference)