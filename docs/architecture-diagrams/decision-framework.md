# Decision Framework

A visual companion to the comparison table in the main [README](../../README.md) and [GUIDE.md](../GUIDE.md). Use this to walk through which strategy fits a given team/workload.

```mermaid
flowchart TD
    Start([Choosing an agent<br/>deployment strategy]) --> Q1{Need a specific<br/>open-source or<br/>custom-tuned model?}

    Q1 -->|Yes| SelfManaged["Self-Managed<br/>(vLLM/Inferentia on EKS)"]
    Q1 -->|No| Q2{Data must stay<br/>strictly inside<br/>your own VPC/infra?}

    Q2 -->|Yes, fully| SelfManaged
    Q2 -->|Partially OK| Q3{Want to own the<br/>agent's orchestration<br/>logic and debug it directly?}

    Q3 -->|Yes| Q4{Willing to run<br/>an EKS cluster for<br/>agent pods + observability?}
    Q3 -->|No, ship fast| FullyManaged["Fully Managed<br/>(Bedrock Agents + AgentCore)"]

    Q4 -->|Yes| Integrated["Integrated<br/>(EKS orchestration +<br/>Bedrock/AgentCore backends)"]
    Q4 -->|No| FullyManaged

    style SelfManaged fill:#e8f4ea,stroke:#2f7a3d
    style Integrated fill:#eaf0fb,stroke:#2f5fa8
    style FullyManaged fill:#fbeeea,stroke:#a8502f
```

## Side-by-side comparison

```mermaid
quadrantChart
    title Control vs. Time-to-Production
    x-axis Low Time-to-Production --> High Time-to-Production
    y-axis Low Control --> High Control
    quadrant-1 High control, slow to ship
    quadrant-2 High control, fast to ship
    quadrant-3 Low control, fast to ship
    quadrant-4 Low control, slow to ship
    Self-Managed: [0.85, 0.9]
    Integrated: [0.5, 0.65]
    Fully Managed: [0.15, 0.3]
```

## Reference table

| Question | Self-Managed | Integrated | Fully Managed |
|---|---|---|---|
| Need a specific open-source model? | Yes | No | No |
| Team has Kubernetes expertise? | Required | Required | Not required |
| Data must stay in your VPC? | Yes | Partial | Partial |
| Time to production matters most? | Slow | Middle | Best |
| Want cloud-portable architecture? | Yes | Partial | No |
| Complex multi-agent workflows? | Full control | Full control | Limited |

See [`docs/GUIDE.md`](../GUIDE.md) for the full cost/ops trade-off breakdown behind this framework.