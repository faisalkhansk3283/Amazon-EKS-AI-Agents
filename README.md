# Amazon EKS AI Agents

Hands-on reference implementation for deploying **agentic AI systems on Amazon EKS** using three deployment strategies: fully self-managed (open source, on Kubernetes), integrated (self-hosted orchestration + AWS managed backends), and fully managed (all AWS services). Built from an AWS Workshop Studio lab, restructured here as a standalone, reusable repo.

> **Audience:** Cloud / GenAI Solutions Architects, ML platform engineers, and anyone evaluating "build vs. buy" trade-offs for agent infrastructure on AWS.

---

## Why this repo exists

Every team building AI agents eventually asks: *"Should we run our own model servers, or use Bedrock? Our own vector DB, or AgentCore Memory? Our own tool protocol, or a managed sandbox?"*

There's no single right answer — it's a **per-component decision**, not an all-or-nothing platform choice. This repo demonstrates the **same customer-service agent** built three different ways, so you can compare real code, real trade-offs, and real operational overhead side by side.

---

## The Three Strategies

| | Self-Managed | Integrated | Fully Managed |
|---|---|---|---|
| **Model inference** | vLLM on EKS (Inferentia/GPU) | Amazon Bedrock (via LiteLLM proxy) | Amazon Bedrock |
| **Agent orchestration** | Strands SDK on EKS | Strands SDK on EKS | Bedrock Agents |
| **Memory** | Milvus (vector DB on EKS) | AgentCore Memory | AgentCore Memory |
| **Tools** | MCP server on EKS | AgentCore Browser / Code Interpreter | AgentCore Gateway, Lambda |
| **Observability** | Langfuse on EKS | Langfuse on EKS | CloudWatch, Bedrock logging |
| **Multi-agent** | A2A protocol | A2A protocol | — |
| **Time to production** | Slowest | Middle | Fastest |
| **Control / flexibility** | Highest | High | Lowest |
| **Ops burden** | Highest (you run everything) | Medium (K8s + managed mix) | Lowest |
| **Best for** | Custom models, data residency, existing K8s teams | Own the agent logic, offload infra-heavy pieces | Ship fast, no custom model needs |

This repo contains full working code + labs for **Self-Managed** and **Integrated**. See [`docs/GUIDE.md`](docs/GUIDE.md) for the full decision framework, including Fully Managed.

---

## Architecture at a Glance

```mermaid
flowchart TB
    subgraph EKS["EKS Cluster (Auto Mode)"]
        direction TB
        Agent["Strands Agent<br/>(same code, both tracks)"]
        LiteLLM["LiteLLM Proxy<br/>(shared model plane)"]
        Langfuse["Langfuse<br/>(observability)"]
        MCP["MCP Server<br/>(custom tools)"]
        Milvus["Milvus<br/>(vector DB — self-managed only)"]
        Neo4j["Neo4j<br/>(knowledge graph — self-managed only)"]
    end

    subgraph SelfHosted["Self-Hosted Backend"]
        vLLM["vLLM on Inferentia<br/>(Qwen2.5-3B)"]
    end

    subgraph AWSManaged["AWS Managed Backend"]
        Bedrock["Amazon Bedrock<br/>(Nova / Claude)"]
        AgentCoreMem["AgentCore Memory"]
        AgentCoreTools["AgentCore Browser +<br/>Code Interpreter"]
    end

    Agent --> LiteLLM
    Agent --> MCP
    Agent --> Milvus
    Agent --> Neo4j
    Agent -.-> AgentCoreMem
    Agent -.-> AgentCoreTools
    LiteLLM --> vLLM
    LiteLLM --> Bedrock
    Agent --> Langfuse
    LiteLLM --> Langfuse
```

**The key idea:** LiteLLM sits as an abstraction layer in front of every model backend. Swapping `model_id="qwen2-5-3b-neuron"` → `model_id="nova-lite"` is often the *only* code change needed to move an agent from self-hosted to Bedrock.

---

## Repo Structure

```
Amazon-EKS-AI-Agents/
├── README.md                        ← you are here
├── LICENSE
├── docs/
│   ├── GUIDE.md                     ← plain-English concept guide + decision framework
│   ├── COMMANDS.md                  ← every command used, organized by lab, copy-paste ready
│   └── architecture-diagrams/       ← standalone Mermaid diagrams per module
│
├── self-managed/                    ← Strategy 1: open-source stack on EKS
│   ├── 100-model-plane/             ← vLLM + Qwen2.5-3B on Inferentia, fronted by LiteLLM
│   ├── 200-strands-agents/          ← core agent loop (Strands SDK)
│   ├── 300-observability-langfuse/  ← OTel tracing → Langfuse
│   ├── 400-rag-milvus/              ← product catalog RAG
│   ├── 500-memory-milvus/           ← session memory (recency-based)
│   ├── 600-agent-tools-mcp/         ← tools exposed over MCP protocol
│   ├── 700-multi-agent-a2a/         ← orchestrator + specialist agents (A2A)
│   ├── 750-evaluation-judge/        ← LLM-as-a-Judge scoring via Langfuse
│   └── 800-knowledge-graph/         ← Neo4j graph traversal tools
│
├── integrated/                      ← Strategy 2: EKS orchestration + AWS managed backends
│   ├── 100-strands-bedrock/         ← same agent, model_id swapped to Bedrock
│   ├── 200-observability-langfuse/  ← same Langfuse, tracing Bedrock calls
│   ├── 300-memory-agentcore/        ← AgentCore Memory (session events)
│   ├── 400-managed-tools/           ← AgentCore Code Interpreter + Browser
│   ├── 500-multi-agent-a2a/         ← A2A specialists riding on Bedrock
│   └── 550-evaluation-agentcore/    ← AgentCore Evaluations (managed LLM-as-a-Judge)
```

Each numbered folder is self-contained: `agent.py`, `tools.py`, `Dockerfile`, `k8s.yaml`, and a lab-specific `README.md` with the exact commands to build/deploy/test it. Copy any single folder out and it should stand alone.

---

## Quickstart

**5-minute read:** Skim the table above + the Mermaid diagram. That's the whole idea.

**Weekend deep-dive:** Start at `self-managed/100-model-plane/`, work top to bottom through `self-managed/`, then repeat the same journey in `integrated/` and diff the two `agent.py` files at each step — the deltas are the lesson.

**If you're running this on AWS Workshop Studio:** the underlying infra (EKS cluster, VPC, LiteLLM proxy, IAM, Milvus/Neo4j/Langfuse Helm releases) is provisioned by the workshop's Terraform and is **not** included in this repo — this repo holds the *application layer* (agent code, tool definitions, Kubernetes manifests, seed scripts) that runs on top of it. See [`docs/COMMANDS.md`](docs/COMMANDS.md) for the full command sequence per module.

---

## Glossary

| Term | Meaning |
|---|---|
| **Strands SDK** | Open-source Python agent framework. The LLM decides which tools to call and in what order. |
| **LiteLLM** | OpenAI-compatible proxy that routes requests to different model backends (vLLM, Bedrock) by `model_id`. Lets agent code stay backend-agnostic. |
| **MCP (Model Context Protocol)** | Open protocol for how agents discover and call tools over the network, decoupling tool logic from agent code. |
| **A2A (Agent-to-Agent)** | Protocol for multi-agent systems — one orchestrator routes requests to specialist agents over JSON-RPC. |
| **AgentCore** | AWS's managed suite for agent memory, sandboxed tool execution (Browser, Code Interpreter), and evaluation. |
| **Langfuse** | Open-source LLM observability platform — ingests OpenTelemetry spans and renders them as structured traces. |
| **RAG** | Retrieval-Augmented Generation — grounding LLM answers in retrieved documents (here, via vector search on Milvus). |
| **Knowledge Graph** | Nodes + typed relationships (Neo4j) for multi-hop questions vector search can't answer, e.g. "what do people who bought X also buy?" |
| **LLM-as-a-Judge** | Using a stronger model to automatically score the quality/accuracy/safety of a weaker (often cheaper) model's outputs. |

---

## Video Walkthrough

Here's a walkthrough of implemented required features:

<img src='https://imgur.com/a/VR4VE4k.gif' title='Video Walkthrough' width='' alt='Video Walkthrough' />

---

## License

See [`LICENSE`](LICENSE).
