# Component Deep Dive: Milvus

**Role in this repo:** self-managed vector database, used for two distinct jobs in the self-managed track — product catalog RAG (module 400) and session conversation memory (module 500).

---

## What it actually is

Milvus is an open-source vector database built for similarity search at scale. Data is stored as high-dimensional vectors (embeddings); queries ask "find me the vectors closest to this one." It supports multiple index types (IVF, HNSW, DiskANN), hybrid search (vector + scalar filters), and scales from single-pod standalone mode to a distributed cluster. This repo runs it in standalone mode (`milvus-standalone`, `etcd`, `minio` pods) — sufficient for a workshop-scale catalog and session store.

## Job 1: RAG over the product catalog (module 400)

**Problem it solves:** "Do you have wireless headphones under $100?" — a fuzzy, semantic question that a keyword or hardcoded lookup can't answer well.

**Pipeline:**
1. Product descriptions and FAQs are embedded once (`seed_products.py`) and stored in a `product_catalog` collection.
2. A customer question hits the `search_products` tool.
3. The tool embeds the query with **fastembed** (a lightweight, ONNX-runtime embedding library — no full ML framework needed in the image) using `all-MiniLM-L6-v2`.
4. Milvus returns the nearest matches; the LLM writes an answer grounded in them.

```python
_embedder = TextEmbedding(model_name="sentence-transformers/all-MiniLM-L6-v2", cache_dir="/app/.fastembed_cache")
_client = MilvusClient(uri=MILVUS_URI)

@tool
def search_products(query: str, limit: int = 5) -> list:
    """Search the AnyCompany Shop product catalog and FAQs.
    Use this when a customer asks about products, pricing, shipping, returns, or warranties.
    """
    embeddings = [v.tolist() for v in _embedder.embed([query])]
    results = _client.search(COLLECTION, data=embeddings, ...)
```

Notes:
- The embedder and Milvus client are **module-level singletons**, loaded once at import and reused across every call — not recreated per request.
- **The Milvus client is not thread-safe.** If you scale the agent to a multi-threaded server, pool the client rather than sharing one instance.
- The `@tool` docstring's first line is the model's *selection heuristic* — the LLM decides whether to call this tool largely based on that one sentence. Write it carefully.

## Job 2: Session conversation memory (module 500)

**Problem it solves:** letting the agent recall "ORD-12345" from turn 1 when the customer asks "has it shipped yet?" in turn 2, without the customer repeating themselves.

This reuses the *same Milvus instance* but a completely different collection (`conversation_memory`) and a completely different access pattern:

| | RAG (module 400) | Memory (module 500) |
|---|---|---|
| What's stored | Product catalog + FAQ embeddings (static, shared) | The customer's own conversation turns (grows over time) |
| Keyed by | Nothing — one shared catalog | `actor_id` + `session_id` for this customer's session |
| Retrieval | Vector search ("find similar products") | Recency ("this session's recent turns, in order") |
| Purpose | Ground answers in product knowledge | Continue the conversation with context |

Crucially, **memory retrieval here is a scalar query, not a vector search**:

```python
def recent_turns(actor_id, session_id, max_results=20):
    """This session's turns, oldest first. Scalar query, no vector search."""
    rows = _client.query(
        COLLECTION,
        filter=f'actor_id == "{actor_id}" && session_id == "{session_id}"',
        output_fields=["user_message", "assistant_message", "ts"],
        limit=max_results, consistency_level="Strong",
    )
```

- `consistency_level="Strong"` is the load-bearing line. Milvus's **default is bounded staleness** — without `Strong`, the turn you just wrote might not be visible to the very next read, silently breaking multi-turn recall.
- Each turn is still stored **as an embedding** even though retrieval here is scalar-only. This is intentional: it means extending to *semantic* cross-session recall later (e.g. "did this customer ever ask about return policies before?") is a one-line change — swap the scalar query for a vector search. No re-architecture, no backfill.

## Deployment

```bash
kubectl get pods -n milvus   # expect milvus-standalone, etcd, minio

# Seed the product catalog (module 400)
IMG=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/customer-agent:milvus
kubectl run milvus-seed --rm -i --restart=Never \
  --image=$IMG --image-pull-policy=Always \
  --env="MILVUS_URI=http://milvus.milvus.svc.cluster.local:19530" \
  --command -- python seed_products.py
```

The seeder runs as a one-shot pod inside the cluster (using the agent's own image) rather than requiring `fastembed`/`pymilvus` installed locally — a pattern worth reusing any time you need a one-off data-loading job against in-cluster infra.

## Common pitfalls

| Symptom | Likely cause |
|---|---|
| Turn 2 doesn't recall turn 1 | Missing `consistency_level="Strong"` on the read, or `actor_id`/`session_id` mismatch between write and read |
| `search_products` never gets called | Tool docstring's first line doesn't clearly signal *when* to use it — the LLM is choosing not to invoke it |
| Milvus client errors under load | Client instance shared across threads without pooling — Milvus's Python client is not thread-safe |
| Seed script "Inserted 0 items" | `MILVUS_URI` pointing at wrong service/namespace, or embedding model failed to download (check `cache_dir` permissions) |

## Related docs

- [`../GUIDE.md`](../GUIDE.md) — section 4 explains the RAG-vs-session-memory distinction conceptually
- [`neo4j.md`](neo4j.md) — the companion "relationships, not similarity" retrieval pattern
- [`agentcore.md`](agentcore.md) — the integrated track's equivalent for session memory (AgentCore Memory replaces Milvus for that job only; RAG still uses Milvus or Bedrock Knowledge Bases)