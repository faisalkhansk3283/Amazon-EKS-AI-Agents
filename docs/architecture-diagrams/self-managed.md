# Self-Managed Architecture

Full open-source stack running on EKS. Every layer — model serving, orchestration, memory, tools, observability — is deployed and operated by you.

```mermaid
flowchart TB
    User([Customer]) --> ChatUI[Chat UI]
    ChatUI --> Agent

    subgraph EKS["EKS Cluster (Auto Mode)"]
        subgraph GeneralNodes["General Purpose Nodes"]
            Agent["Strands Agent<br/>(customer-agent pod)"]
            Langfuse["Langfuse<br/>(observability)"]
            Milvus[("Milvus<br/>vector DB")]
            Neo4j[("Neo4j<br/>knowledge graph")]
            MCPServer["MCP Server<br/>(lookup_order, check_inventory,<br/>initiate_return)"]
        end

        subgraph InferentiaPool["Inferentia Node Pool"]
            vLLM["vLLM<br/>Qwen2.5-3B on Neuron"]
        end

        subgraph LiteLLMNS["litellm namespace"]
            LiteLLM["LiteLLM Proxy<br/>OpenAI-compatible"]
        end
    end

    Agent -->|"OpenAIModel<br/>base_url=litellm:4000"| LiteLLM
    LiteLLM -->|model_id=qwen2-5-3b-neuron| vLLM
    Agent -->|tool calls| MCPServer
    Agent -->|vector search| Milvus
    Agent -->|graph traversal| Neo4j
    Agent -->|OTel spans| Langfuse
    LiteLLM -->|proxy spans| Langfuse

    subgraph A2A["Multi-Agent (A2A protocol)"]
        Orchestrator["Orchestrator Agent"]
        OrderAgent["Order Agent"]
        ProductAgent["Product Agent"]
    end

    ChatUI -.->|Multi-Agent mode| Orchestrator
    Orchestrator -->|A2A JSON-RPC| OrderAgent
    Orchestrator -->|A2A JSON-RPC| ProductAgent
    OrderAgent --> MCPServer
    ProductAgent --> Milvus
```

## Key characteristics

- **One model plane**: LiteLLM fronts vLLM so the agent never talks to the model server directly.
- **Everything in-cluster**: no external AWS managed AI services in the request path.
- **Full trace visibility**: Langfuse captures both the agent's own OTel spans and LiteLLM's proxy-level spans, so agent-side and inference-side latency can be told apart.
- **Ops cost**: you own patching, scaling, and reliability for every box in this diagram.