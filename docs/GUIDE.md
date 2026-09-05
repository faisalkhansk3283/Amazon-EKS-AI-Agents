# Guide: Understanding Agentic AI Patterns on AWS

This document explains the concepts behind this repo in plain language — no assumed familiarity with agent frameworks or AWS's newer GenAI services. Read this before diving into the code if any of the acronyms in the README felt unfamiliar.

## 1. What is an "agent," really?

A traditional LLM call is one-shot: you send a prompt, you get text back. An **agent** adds a loop:

1. The LLM receives a user query plus a list of **tools** it's allowed to call (e.g. "look up an order," "search products").
2. The LLM decides: answer directly, or call a tool first?
3. If it calls a tool, the *code* (not the LLM) actually runs it, and the result is fed back to the LLM.
4. The LLM repeats step 2 until it has enough information to answer.

That's it. Everything else in this repo — memory, observability, multi-agent orchestration — is scaffolding around that loop.

## 2. The three deployment strategies, explained simply

Imagine you're building a customer-service chatbot for an online store. It needs to: understand language (a model), remember the conversation (memory), look things up (tools), and you need to know when it's wrong (observability). For each of these four needs, you have a choice: **run it yourself** or **rent it from AWS**.

- **Self-Managed** = run everything yourself, on your own Kubernetes cluster, using open-source software. Maximum control, maximum work.
- **Fully Managed** = rent everything from AWS (Bedrock for the model, Bedrock Agents for orchestration, AgentCore for memory/tools). Minimum work, but you're boxed into what AWS offers.
- **Integrated** = a middle ground. You keep the agent *code* and orchestration logic on your own EKS cluster (so you can debug and customize the actual decision-making), but you rent the heavy infrastructure pieces — model hosting, memory storage, sandboxed tool execution — from AWS.

Most real teams land on **Integrated**: full control over the part that differentiates their product (the agent's reasoning and tools), no interest in babysitting GPU fleets or vector database clusters.

## 3. Why LiteLLM matters

LiteLLM is a small but important idea: it's a proxy server that speaks the same API format (OpenAI's) regardless of what's actually generating the text behind it — could be a self-hosted open-source model on your own GPUs, could be Amazon Bedrock, could be another cloud entirely.

Your agent code talks to LiteLLM once, and never has to know or care which backend served the request. Switching backends becomes a **one-line config change** (`model_id="qwen2-5-3b-neuron"` → `model_id="nova-lite"`), not a rewrite. This repo's whole "self-managed vs. integrated" comparison exists because of this abstraction — the agent code is almost identical in both tracks.

## 4. Memory: two different problems people conflate

"Memory" for an agent actually covers two unrelated things:

- **Session memory** — remembering what the customer said five messages ago in *this* conversation, so they don't have to repeat their order number. This is short-lived, per-conversation, and retrieved by recency (just get the last N turns, in order).
- **Knowledge retrieval (RAG)** — searching a large, mostly-static body of information (a product catalog, FAQ, documentation) to find text *relevant* to the current question. This is retrieved by semantic similarity ("find things like this"), not recency.

In this repo: Milvus is used for **both** in the self-managed track (different collections, different query patterns), while the integrated track splits them — AgentCore Memory handles session memory, and Milvus (or Bedrock Knowledge Bases) still handles RAG.

## 5. Tools: MCP vs. AgentCore sandboxes

Two different tool problems:

- **Business-logic tools** (look up an order, check inventory) are specific to your domain. Nobody at AWS can pre-build these for you. **MCP (Model Context Protocol)** is just a standard way to expose these tools over the network so any agent can discover and call them, instead of hardcoding function calls into the agent's source file.
- **Generic risky tools** (run arbitrary Python code, fetch a webpage) are the same for everyone, but dangerous to run inside your own infrastructure — arbitrary code execution next to your production pods is a real risk. **AgentCore Code Interpreter / Browser** are AWS-managed, isolated sandboxes for exactly this class of tool.

Rule of thumb used throughout this repo: **domain tools → MCP on your own infra. Generic/risky tools → managed sandboxes.**

## 6. Multi-agent systems (A2A)

A single agent with a huge system prompt and 15 tools starts making mistakes — it confuses similar tools, and every change risks breaking something unrelated. The fix: split into **specialist agents**, each with one job and a short, focused system prompt (an "Order Agent," a "Product Agent"), and add an **orchestrator** that reads the user's query and routes it to the right specialist.

**A2A (Agent-to-Agent protocol)** is the wire format specialists use to talk to the orchestrator and each other — each specialist runs as its own small HTTP service and publishes an "agent card" describing what it can do, so the orchestrator can discover and call it like a tool.

## 7. Observability: why you can't debug an agent by reading its final answer

An agent's wrong answer could come from: the LLM misunderstanding the question, calling the wrong tool, a tool returning bad data, or the LLM ignoring correct tool data and hallucinating anyway. Without step-by-step visibility, you can't tell which.

**Langfuse** ingests OpenTelemetry traces and renders them as a tree: one span per LLM call, one span per tool call, nested under the overall conversation turn. You can see exactly what was sent to the model, what came back, which tool ran, and how long each step took.

**AgentCore Observability** is AWS's managed equivalent, reading the same kind of OpenTelemetry data but through CloudWatch/X-Ray instead of a self-hosted Langfuse instance.

## 8. Evaluation: grading the agent automatically

Reading agent conversations by hand doesn't scale. **LLM-as-a-Judge** uses a *stronger* model (e.g. Claude Sonnet) to grade the *cheaper* model's (e.g. a 3B parameter model, or Nova Lite) actual conversations against criteria you write — accuracy, helpfulness, safety — producing a 0–1 score with reasoning, attached back onto the trace.

Two implementations appear in this repo:
- **Self-managed:** Langfuse's built-in evaluator framework, calling out to a judge model through LiteLLM.
- **Integrated:** **AgentCore Evaluations**, AWS's managed evaluation service, with 16 pre-built evaluators (correctness, helpfulness, tool selection, safety, etc.) plus support for custom ones.

Important finding replicated in both labs: **a judge can only grade what your instrumentation captured.** If your traces don't record the tool-call span, the judge can't verify grounding and will (correctly) mark answers as unverifiable/low-accuracy — even if the agent's answer was actually correct. Rich tracing isn't optional if you want evaluation to mean anything.

## 9. RAG vs. Knowledge Graphs — different questions, not competing tools

- **Vector search (RAG)** answers: *"what text is similar to this?"* — good for fuzzy, unstructured matching like "wireless headphones under $100."
- **Knowledge graphs (Neo4j)** answer: *"what is connected to this thing, and how?"* — good for precise, multi-hop questions like "what do people who bought this laptop usually also buy?" (a 4-hop traversal: product → orders → customers → their other orders → products).

Production systems often combine both (sometimes called GraphRAG): vector search finds the starting point, the graph expands outward from it. Neither replaces the other.

## 10. Decision Framework (reference table)

| Question | Self-Managed | Integrated | Fully Managed |
|---|---|---|---|
| Need a specific open-source model? | Yes | No | No |
| Team has Kubernetes expertise? | Required | Required | Not required |
| Data must stay in your VPC? | Yes | Partial | Partial |
| Time to production matters most? | Slow | Middle | Best |
| Want cloud-portable architecture? | Yes | Partial | No |
| Complex multi-agent workflows? | Full control | Full control | Limited |

## 11. Cost & Ops Trade-offs (at a glance)

| | Self-Managed | Integrated | Fully Managed |
|---|---|---|---|
| **Idle cost** | You pay for reserved GPU/Inferentia capacity even when idle | Pay-per-use on Bedrock; EKS control plane + node cost for orchestration only | Pay-per-use everywhere |
| **Who patches what** | You patch OS, K8s, model server, every dependency | You patch K8s + agent pods; AWS patches Bedrock/AgentCore | AWS patches everything |
| **Debugging depth** | Full stack trace access, but you built the stack | Full agent-logic visibility, opaque managed backends | Orchestration logic is opaque — hardest to debug complex flows |
| **Scaling** | Manual/Karpenter-based, you own capacity planning | Bedrock auto-scales; EKS still needs node scaling for agent pods | Fully automatic |
| **Vendor lock-in risk** | Lowest — portable to any Kubernetes | Medium — agent code portable, backends AWS-specific | Highest — Bedrock Agents orchestration logic isn't portable |

---

**Next:** see [`COMMANDS.md`](COMMANDS.md) for the exact command sequence to build and run any module in this repo, or dive into the numbered folders under `self-managed/` and `integrated/` for full source code and per-module READMEs.