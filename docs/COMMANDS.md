# Command Reference

All commands used across every lab in this repo, organized by module. Copy-paste ready. Assumes you're running from an AWS Workshop Studio IDE terminal (or any shell with `kubectl`, `aws`, `docker`, and `envsubst` configured against the workshop's EKS cluster).

**Environment variables used throughout** (typically pre-exported by the workshop IDE — verify with `echo $VAR_NAME`):

```bash
echo $ACCOUNT_ID
echo $AWS_REGION
echo $CHAT_UI_URL
```

---

## Self-Managed Track

### 100 — Model Plane (vLLM + LiteLLM)
No agent deployment in this module — infra is pre-provisioned by Terraform. Verify it's up:

```bash
kubectl get pods -n litellm
kubectl get pods -n vllm
```

### 200 — Strands Agents
```bash
cd ~/environment/modules/20-self-managed/200-strands-agents/customer-agent

# Build and push the image
IMG=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/customer-agent:strands
docker build --push -t $IMG .

# Deploy
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent --timeout=120s
```

### 300 — Observability (Langfuse)
```bash
kubectl get pods -n langfuse

echo "Langfuse UI: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
# Login: admin@workshop.local / workshop2025

cd ~/environment/modules/20-self-managed/300-observability-langfuse/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent --timeout=120s
```

### 400 — RAG with Milvus
```bash
kubectl get pods -n milvus

# Seed the product catalog
IMG=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/customer-agent:milvus
kubectl run milvus-seed --rm -i --restart=Never \
  --image=$IMG \
  --image-pull-policy=Always \
  --env="MILVUS_URI=http://milvus.milvus.svc.cluster.local:19530" \
  --command -- python seed_products.py

cd ~/environment/modules/20-self-managed/400-rag-milvus/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent --timeout=180s
```

### 500 — Memory Management (Milvus)
```bash
kubectl get pods -n milvus

cd ~/environment/modules/20-self-managed/500-memory-milvus/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent --timeout=180s
```

### 600 — Agent Tool Access (MCP)
```bash
# Deploy MCP server
cd ~/environment/modules/20-self-managed/600-agent-tools-mcp/mcp-server
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/mcp-server --timeout=60s

# Deploy agent connected to it
cd ~/environment/modules/20-self-managed/600-agent-tools-mcp/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent --timeout=180s
```

### 700 — Multi-Agent (A2A)
```bash
cd ~/environment/modules/20-self-managed/700-multi-agent-a2a/a2a-agents

envsubst < k8s-specialists.yaml  | kubectl apply -f -
envsubst < k8s-orchestrator.yaml | kubectl apply -f -

kubectl rollout status deployment/order-agent        --timeout=120s
kubectl rollout status deployment/product-agent      --timeout=120s
kubectl rollout status deployment/orchestrator-agent --timeout=120s
```

### 750 — Evaluation (LLM-as-a-Judge, Langfuse)
```bash
# Verify LiteLLM host is allowlisted in Langfuse (SSRF guard)
for d in langfuse-web langfuse-worker; do
  for var in LANGFUSE_LLM_CONNECTION_WHITELISTED_HOST ENCRYPTION_KEY; do
    echo -n "$d / $var: "
    kubectl get deploy "$d" -n langfuse \
      -o jsonpath="{.spec.template.spec.containers[0].env[?(@.name=='$var')].value}{'\n'}"
  done
done

# If allowlist is missing:
kubectl set env deployment/langfuse-web deployment/langfuse-worker -n langfuse \
  LANGFUSE_LLM_CONNECTION_WHITELISTED_HOST=litellm.litellm.svc.cluster.local
kubectl rollout status deployment/langfuse-web -n langfuse
kubectl rollout status deployment/langfuse-worker -n langfuse
```
Evaluators (`cs-accuracy`, `cs-safety`, `Helpfulness`) are created in the Langfuse UI — no CLI for this step. See the module's own README for exact prompts and mapping.

### 800 — Knowledge Graph (Neo4j)
```bash
kubectl get pods -n neo4j

export NEO4J_PASSWORD=$(kubectl get configmap agent-config -o jsonpath='{.data.NEO4J_PASSWORD}')
echo $NEO4J_PASSWORD

IMG=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/customer-agent:graph
kubectl run graph-seed --rm -i --restart=Never \
  --image=$IMG \
  --image-pull-policy=Always \
  --env="NEO4J_URI=neo4j://neo4j.neo4j.svc.cluster.local:7687" \
  --env="NEO4J_PASSWORD=$NEO4J_PASSWORD" \
  --command -- python seed_graph.py

echo "Neo4j Browser: http://$(kubectl get svc -n neo4j neo4j-lb-neo4j -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'):7474"
echo "Username: neo4j"
echo "Password: $NEO4J_PASSWORD"

cd ~/environment/modules/20-self-managed/800-knowledge-graph/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent --timeout=180s
```

### Self-Managed Cleanup
```bash
kubectl delete deployment customer-agent order-agent product-agent orchestrator-agent mcp-server 2>/dev/null
kubectl delete service customer-agent order-agent product-agent orchestrator-agent mcp-server 2>/dev/null
kill $(lsof -t -i:8000) 2>/dev/null
```

---

## Integrated Track

### 100 — Strands with Bedrock
```bash
cd ~/environment/modules/30-integrated/100-strands-bedrock/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent -n agents --timeout=120s

echo $CHAT_UI_URL
```

### 200 — Observability (Langfuse, Bedrock-backed)
```bash
echo "Langfuse UI: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

cd ~/environment/modules/30-integrated/200-observability-langfuse/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent -n agents --timeout=120s
```

### 300 — Memory Management (AgentCore Memory)
```bash
kubectl get configmap agent-config -n agents -o yaml | grep AGENTCORE

cd ~/environment/modules/30-integrated/300-memory-agentcore/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent -n agents --timeout=120s

# Inspect stored events
MEMORY_ID=$(kubectl get cm agent-config -n agents -o jsonpath='{.data.AGENTCORE_MEMORY_ID}')
SESSION_ID=$(kubectl logs -n agents deployment/customer-agent --tail=200 | grep -oE 'ui-[a-f0-9-]+' | tail -1)

aws bedrock-agentcore list-events \
  --memory-id "$MEMORY_ID" \
  --actor-id "workshop-user" \
  --session-id "$SESSION_ID" \
  --include-payloads
```

### 400 — Managed Tools (AgentCore Browser + Code Interpreter)
```bash
kubectl get configmap agent-tools -n agents -o yaml

cd ~/environment/modules/30-integrated/400-managed-tools/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent -n agents --timeout=180s
```

### 500 — Multi-Agent (A2A on Bedrock)
```bash
cd ~/environment/modules/30-integrated/500-multi-agent-a2a/a2a-integrated

envsubst < k8s-specialists.yaml  | kubectl apply -f -
envsubst < k8s-orchestrator.yaml | kubectl apply -f -

kubectl rollout status deployment/mcp-server          -n agents --timeout=120s
kubectl rollout status deployment/order-agent         -n agents --timeout=120s
kubectl rollout status deployment/sandbox-agent       -n agents --timeout=120s
kubectl rollout status deployment/orchestrator-agent  -n agents --timeout=120s
```

### 550 — Evaluation (AgentCore Evaluations)
```bash
# Confirm prerequisites
kubectl get configmap agent-config -n agents -o jsonpath='{.data.AGENTCORE_LOG_GROUP}'; echo
kubectl get configmap agent-config -n agents -o jsonpath='{.data.AGENTCORE_EVAL_ROLE_ARN}'; echo

# Deploy
cd ~/environment/modules/30-integrated/550-evaluation-agentcore/customer-agent
envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent -n agents --timeout=120s

# Confirm traces landed (allow 60-90s indexing lag)
START=$(( ($(date +%s) - 600) * 1000 ))
aws logs filter-log-events --region $AWS_REGION --log-group-name "aws/spans" \
  --start-time "$START" --max-items 200 --query 'events[].message' --output json \
  | jq '[ .[] | fromjson | select(.resource.attributes."service.name" == "customer-agent-eval") ] | length'

# List built-in evaluators
aws bedrock-agentcore-control list-evaluators --region $AWS_REGION \
  --query "evaluators[?evaluatorType=='Builtin'].evaluatorId" --output text

# Create custom evaluator (config file — see module README for full JSON)
CUSTOM_ID=$(aws bedrock-agentcore-control create-evaluator \
  --region $AWS_REGION \
  --evaluator-name cs_accuracy \
  --level TRACE \
  --description "Retail order-accuracy evaluator (tool-grounded)" \
  --evaluator-config file:///tmp/cs_accuracy_config.json \
  --query 'evaluatorId' --output text)
echo "Created custom evaluator: $CUSTOM_ID"

# Get session to score
SESSION_ID=$(kubectl logs -n agents deployment/customer-agent --tail=100 \
  | grep -oE 'session=[^ ]+' | tail -1 | cut -d= -f2)

# Fetch session spans
START=$(( ($(date +%s) - 3600) * 1000 ))
aws logs filter-log-events --region $AWS_REGION --log-group-name "aws/spans" \
  --start-time "$START" --max-items 300 --query 'events[].message' --output json \
  | jq --arg s "$SESSION_ID" \
      '{ sessionSpans: [ .[] | fromjson | select(.attributes."session.id" == $s) ] }' \
  > /tmp/eval_input.json

TRACE_ID=$(jq -r '.sessionSpans[0].traceId' /tmp/eval_input.json)

# Run each evaluator
for E in "Builtin.Correctness" "Builtin.Helpfulness" "$CUSTOM_ID"; do
  aws bedrock-agentcore evaluate --region $AWS_REGION \
    --evaluator-id "$E" \
    --evaluation-input  file:///tmp/eval_input.json \
    --evaluation-target "{\"traceIds\":[\"$TRACE_ID\"]}" \
    --query 'evaluationResults[0].{evaluator:evaluatorName,score:value,label:label,reason:explanation}' \
    --output json
done
```

### Integrated Cleanup
```bash
kubectl delete deployment orchestrator-agent order-agent sandbox-agent mcp-server -n agents 2>/dev/null
kubectl delete service orchestrator-agent order-agent sandbox-agent mcp-server -n agents 2>/dev/null

kubectl delete deployment customer-agent -n agents 2>/dev/null
kubectl delete service customer-agent -n agents 2>/dev/null

kubectl get deployments,services -n agents   # verify only agent SA + ConfigMaps remain

# Optional: remove custom evaluator
CUSTOM_ID=$(aws bedrock-agentcore-control list-evaluators --region $AWS_REGION \
  --query "evaluators[?evaluatorType=='Custom'].evaluatorId | [0]" --output text)
aws bedrock-agentcore-control delete-evaluator --evaluator-id "$CUSTOM_ID" --region $AWS_REGION
```

---

## Useful chat-UI test messages (reusable across modules)

```text
Hi, I ordered a laptop last week and it still hasn't arrived. My order ID is ORD-12345. Can you help?
I want to return the headphones I bought. Order ORD-11111.
Can you check on order ORD-99999?
What's your return policy?
Do you have any noise cancelling headphones?
Is the Laptop Pro under warranty?
Where is my order ORD-12345?
Has it shipped yet?
What do people who bought the Laptop Pro 15 usually buy with it?
What has Jane Doe ordered before?
I want to total up what I spent on orders ORD-12345 and ORD-67890.
```