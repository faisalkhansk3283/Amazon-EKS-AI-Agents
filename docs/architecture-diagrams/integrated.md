# Integrated Architecture

Agent code and orchestration stay on EKS; model inference, memory, and risky tool execution move to AWS managed services. LiteLLM is the pivot point — same proxy, same protocol, different backend.

```mermaid
flowchart TB
    User([Customer]) --> ChatUI[Chat UI]
    ChatUI --> Agent

    subgraph EKS["EKS Cluster (Auto Mode)"]
        subgraph AgentsNS["agents namespace"]
            Agent["Strands Agent<br/>(ServiceAccount: agent)<br/>model_id=nova-lite"]
            MCPServer["MCP Server<br/>(domain tools, unchanged from<br/>self-managed track)"]
        end

        subgraph LiteLLMNS["litellm namespace"]
            LiteLLM["LiteLLM Proxy<br/>Pod Identity → Bedrock IAM"]
        end

        subgraph LangfuseNS["langfuse namespace"]
            Langfuse["Langfuse<br/>(reused from self-managed track)"]
        end
    end

    subgraph AWS["AWS Managed Services"]
        Bedrock["Amazon Bedrock<br/>Nova / Claude"]
        AgentCoreMemory["AgentCore Memory<br/>(session events)"]
        AgentCoreBrowser["AgentCore Browser<br/>(sandboxed web actions)"]
        AgentCoreCode["AgentCore Code Interpreter<br/>(sandboxed Python)"]
    end

    Agent -->|"OpenAIModel<br/>base_url=litellm:4000"| LiteLLM
    LiteLLM -->|"Pod Identity<br/>(no static creds)"| Bedrock
    Agent -->|"boto3<br/>(Pod Identity)"| AgentCoreMemory
    Agent -->|"boto3<br/>(Pod Identity)"| AgentCoreBrowser
    Agent -->|"boto3<br/>(Pod Identity)"| AgentCoreCode
    Agent -->|domain tool calls| MCPServer
    Agent -->|OTel spans| Langfuse
    LiteLLM -->|proxy spans| Langfuse

    subgraph A2A["Multi-Agent (A2A protocol)"]
        Orchestrator["Orchestrator Agent<br/>+ AgentCore Memory"]
        OrderAgent["Order Agent<br/>(MCP)"]
        SandboxAgent["Sandbox Agent<br/>(Code Interpreter + Browser)"]
    end

    ChatUI -.->|Multi-Agent mode| Orchestrator
    Orchestrator -->|A2A JSON-RPC| OrderAgent
    Orchestrator -->|A2A JSON-RPC| SandboxAgent
```

## Key characteristics

- **One-line backend swap**: `model_id="qwen2-5-3b-neuron"` → `model_id="nova-lite"` is the only agent-side change needed to move from vLLM to Bedrock — same `OpenAIModel` client, same LiteLLM proxy.
- **Credential-free agent pods**: Bedrock IAM lives on the LiteLLM pod (Pod Identity); AgentCore permissions live on the agent's own ServiceAccount. No static keys anywhere.
- **Domain tools stay on MCP**: business logic (order lookup, inventory) is yours — it stays on EKS. Generic/risky tools (arbitrary code execution, web fetch) move to AgentCore's isolated sandboxes.
- **Observability unchanged**: same Langfuse instance, same OTel span shape — only the `model` label in traces changes (`qwen2-5-3b-neuron` vs `nova-lite`).