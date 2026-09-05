# Component Deep Dives

Each file here is a standalone reference for one piece of the stack — what it is, why it's used, exact code patterns from this repo, deployment commands, and a troubleshooting table. Read [`../GUIDE.md`](../GUIDE.md) first for the conceptual overview; come here for implementation-level detail on a specific component.

| Component | Used in | Covers |
|---|---|---|
| [`litellm.md`](litellm.md) | Both tracks | The shared model-routing proxy; why one `model_id` change moves an agent between backends |
| [`langfuse.md`](langfuse.md) | Both tracks | OTel tracing, trace structure, SSRF/evaluator setup gotchas |
| [`milvus.md`](milvus.md) | Self-managed | Vector DB used for both product RAG and session memory (two collections, two access patterns) |
| [`neo4j.md`](neo4j.md) | Self-managed | Knowledge graph for multi-hop questions; ontology and fixed-Cypher tool design |
| [`mcp.md`](mcp.md) | Both tracks | Domain-tool protocol; server/client pattern; dynamic tool discovery |
| [`a2a.md`](a2a.md) | Both tracks | Multi-agent orchestration; specialists, orchestrator, tracing across process boundaries |
| [`agentcore.md`](agentcore.md) | Integrated | AgentCore Memory, Browser, and Code Interpreter — the managed counterparts to Milvus memory and self-built tools |
| [`evaluation.md`](evaluation.md) | Both tracks | LLM-as-a-Judge, compared side by side — Langfuse-based vs. AgentCore Evaluations |

## Reading order suggestions

**If you're deciding what to build first:** `litellm.md` → `mcp.md` → `langfuse.md`. These three cover the pieces present in nearly every module.

**If you're comparing self-managed vs. integrated for a specific capability:**
- Memory → `milvus.md` (self-managed) + the AgentCore Memory section of `agentcore.md` (integrated)
- Tools → `mcp.md` (shared) + the Browser/Code Interpreter section of `agentcore.md` (integrated-only)
- Evaluation → `evaluation.md` (covers both in one file, side by side)

**If you're building the knowledge-graph or multi-agent modules specifically:** `neo4j.md` and `a2a.md` are mostly self-contained but reference `mcp.md` and `milvus.md` for the tools each specialist wraps.