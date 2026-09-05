# Component Deep Dive: Neo4j

**Role in this repo:** knowledge graph backend (module 800, self-managed track only) — answers multi-hop relationship questions that vector search structurally cannot.

---

## What it actually is

Neo4j is a graph database: data is stored as **nodes** and **typed relationships** rather than rows or embeddings. Queries are written in Cypher, a pattern-matching language ("match this shape in the graph") rather than a similarity search.

## Why it exists alongside Milvus, not instead of it

Both are "retrieval," but they answer fundamentally different kinds of questions:

| | RAG with Milvus | Knowledge graph (Neo4j) |
|---|---|---|
| Data shape | Embeddings (points in vector space) | Nodes + typed relationships |
| Query | "What is similar to this text?" | "What is connected to this thing, and how?" |
| Strength | Fuzzy matching, unstructured text | Multi-hop questions, precise joins |
| Example | "noise cancelling headphones" → product description | "customers who bought X also bought…" → 4-hop traversal |

Neither replaces the other. Production systems often combine them (sometimes called **GraphRAG**): vector search finds the entry-point entity, the graph expands outward from it.

## The ontology

A knowledge graph without an explicit schema drifts into unstructured mush fast. This module constrains itself to **five node types and four relationship types**:

```
(:Customer)-[:PLACED]->(:Order)-[:CONTAINS {qty}]->(:Product)
(:Product)-[:IN_CATEGORY]->(:Category)-[:HAS_POLICY]->(:Policy)
```

Same underlying shop data as every other lab (the orders from the MCP module, the products from the Milvus catalog) plus a handful of historical orders so traversals have somewhere to go. What's new isn't the data — it's that relationships between records are now first-class and queryable, instead of implicit foreign keys.

## The four tools — each a fixed Cypher traversal

A key design decision: the LLM **never writes Cypher**. Each tool is one pre-written, parameterized traversal; the model only picks the tool and fills in arguments. This is what keeps a 3B-parameter model reliable — text-to-Cypher (letting the LLM generate arbitrary graph queries) is a natural extension, but a much harder reliability problem, and this repo deliberately doesn't attempt it in the base lab.

| Tool | Traversal |
|---|---|
| `lookup_order` | order → items → customer (same lookup as MCP module, now a graph walk) |
| `customer_history` | customer → all orders → items |
| `recommend_products` | product → orders containing it → those customers → their other orders → co-purchased products (**4 hops**) |
| `product_policies` | product → category → policies |

The star of the lab is `recommend_products` — an answer that doesn't exist in any single record, only in the connections between records:

```cypher
MATCH (p:Product)<-[:CONTAINS]-(:Order)<-[:PLACED]-(c:Customer)
      -[:PLACED]->(:Order)-[:CONTAINS]->(rec:Product)
WHERE toLower(p.name) CONTAINS toLower($name) AND rec <> p
RETURN rec.name AS product, rec.price AS price, count(DISTINCT c) AS bought_by
ORDER BY bought_by DESC, product
```

Try writing that as a vector search — you can't. It's a relationship question, not a similarity question.

## Implementation notes

- The `neo4j` Python driver is **module-level and thread-safe with built-in connection pooling** — unlike the Milvus client, no manual pooling caveat needed here.
- Query parameters (`$name`) are always passed separately, never string-interpolated into the Cypher text — the same injection-prevention discipline as parameterized SQL.
- No embedding model is needed for this module at all — the Dockerfile drops back to a simple single-stage build (compare to the RAG module's multi-stage build for bundling `fastembed` weights).

## Deployment and seeding

```bash
kubectl get pods -n neo4j   # expect neo4j-0

export NEO4J_PASSWORD=$(kubectl get configmap agent-config -o jsonpath='{.data.NEO4J_PASSWORD}')

IMG=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/customer-agent:graph
kubectl run graph-seed --rm -i --restart=Never \
  --image=$IMG --image-pull-policy=Always \
  --env="NEO4J_URI=neo4j://neo4j.neo4j.svc.cluster.local:7687" \
  --env="NEO4J_PASSWORD=$NEO4J_PASSWORD" \
  --command -- python seed_graph.py
```

## Exploring visually (Neo4j Browser)

Unlike every other data store in this repo, Neo4j ships with a genuinely useful visual explorer:

```bash
echo "Neo4j Browser: http://$(kubectl get svc -n neo4j neo4j-lb-neo4j -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'):7474"
```

```cypher
MATCH (c:Customer)-[r1:PLACED]->(o:Order)-[r2:CONTAINS]->(p:Product)
RETURN c, r1, o, r2, p
```

This renders customers, orders, and products as draggable, clickable nodes — genuinely useful for building intuition about what the agent is actually traversing, in a way JSON output never quite conveys.

## Common pitfalls

| Symptom | Likely cause |
|---|---|
| `recommend_products` returns nothing | Product name doesn't match any node (`toLower(...) CONTAINS toLower($name)` is substring match, not fuzzy — check exact seeded names) |
| Seed script errors on relationship creation | Nodes referenced by a relationship weren't created first — Cypher `MERGE` order matters |
| Traversal is technically correct but slow | Missing an index/constraint on a frequently-matched property (e.g. `Order.id`) — add one via `CREATE INDEX` |
| Agent tries to write its own Cypher and fails | System prompt didn't clearly scope the model to the four fixed tools — check the prompt hasn't drifted toward "write a query for X" |

## Related docs

- [`milvus.md`](milvus.md) — the vector-search counterpart; read both together to internalize the RAG-vs-graph distinction
- [`../GUIDE.md`](../GUIDE.md) — section 9 covers this at a conceptual level