# Ralph Method: agentgateway POC Guide

> **The Ralph Method**: A continuous AI-driven development loop that builds software through iterative refinement.
>
> ```bash
> while :; do cat PROMPT.md | claude ; done
> ```
>
> *"That's the beauty of Ralph - the technique is deterministically bad in an undeterministic world."*
> — [Geoffrey Huntley](https://ghuntley.com/ralph/)

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Project Structure](#project-structure)
4. [Quick Start](#quick-start)
5. [The PROMPT.md Template](#the-promptmd-template)
6. [Running Ralph](#running-ralph)
7. [Tuning Ralph (Adding Signs)](#tuning-ralph-adding-signs)
8. [Example Workflows](#example-workflows)
9. [Troubleshooting](#troubleshooting)
10. [References](#references)

---

## Overview

This guide walks you through using the **Ralph Method** with **Claude Code** to build and test an **agentgateway** Proof of Concept on a local Kind cluster.

### What is agentgateway?

[Agentgateway](https://agentgateway.dev/) is an open source, AI-native gateway built in Rust that provides:

- **LLM Consumption** — Proxy and manage traffic to LLM providers (OpenAI, Anthropic, Gemini, Bedrock)
- **MCP Connectivity** — Federation of Model Context Protocol tool servers
- **Agent Connectivity (A2A)** — Agent-to-agent communication routing
- **Inference Routing** — Kubernetes Gateway API Inference Extension support
- **CEL-based RBAC** — Fine-grained access control for tools and agents

### Why Ralph Works for agentgateway

| Ralph Strength | agentgateway Fit |
|----------------|------------------|
| Greenfield iteration | Gateway/HTTPRoute/Backend configs iterate cleanly |
| Predictable failures | YAML errors, CRD mismatches are learnable |
| Self-testing loop | `kubectl apply`, `curl`, MCP Inspector provide feedback |
| Eventual consistency | Multi-resource AI deployments converge |

---

## Prerequisites

### Required Tools

```bash
# Check if tools are installed
command -v docker && echo "✓ Docker"
command -v kind && echo "✓ Kind"
command -v kubectl && echo "✓ kubectl"
command -v helm && echo "✓ Helm"
command -v claude && echo "✓ Claude Code"
command -v curl && echo "✓ curl"
command -v jq && echo "✓ jq"
```

### Install Missing Tools

```bash
# Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-$(uname | tr '[:upper:]' '[:lower:]')-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Claude Code (Native installation - recommended)
curl -fsSL https://claude.ai/install.sh | bash
source ~/.bashrc  # or source ~/.zshrc

# Verify Claude Code
claude --version
claude doctor

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# jq
sudo apt-get install jq  # or brew install jq on macOS
```

### API Keys (for LLM routing tests)

```bash
# Optional: For testing LLM routing through agentgateway
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
```

---

## Project Structure

```
agentgateway-poc/
├── PROMPT.md              # Ralph's instructions (the brain)
├── CHANGELOG.md           # Track what Ralph builds
├── SIGNS.md               # Learnings from Ralph's mistakes
├── kind/
│   └── cluster-config.yaml
├── manifests/
│   ├── gateway.yaml
│   ├── httproute.yaml
│   ├── backends/
│   │   ├── openai.yaml
│   │   └── mcp-server.yaml
│   └── policies/
├── helm/
│   └── values.yaml
├── scripts/
│   ├── setup.sh
│   ├── test.sh
│   └── teardown.sh
└── tests/
    ├── smoke-test.sh
    └── curl-tests/
```

---

## Quick Start

### 1. Create Project Directory

```bash
mkdir -p agentgateway-poc && cd agentgateway-poc
```

### 2. Create Kind Cluster

```bash
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: agentgateway-poc
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
      - containerPort: 30443
        hostPort: 30443
        protocol: TCP
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
EOF

kind create cluster --config kind-config.yaml
```

### 3. Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

### 4. Install agentgateway

```bash
# Install agentgateway CRDs
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v2.2.0-main \
  agentgateway-crds oci://ghcr.io/kgateway-dev/charts/agentgateway-crds

# Install agentgateway control plane
helm upgrade -i -n agentgateway-system \
  agentgateway oci://ghcr.io/kgateway-dev/charts/agentgateway \
  --version v2.2.0-main \
  --set controller.image.pullPolicy=Always

# Verify installation
kubectl get pods -n agentgateway-system
kubectl get gatewayclass agentgateway
```

### 5. Create PROMPT.md and Start Ralph

```bash
cat > PROMPT.md << 'EOF'
# agentgateway POC

You are building a Proof of Concept for agentgateway on a Kind cluster.

## Current State
- [x] Kind cluster created
- [x] Gateway API CRDs installed (v1.4.0)
- [x] agentgateway installed via Helm (v2.2.0-main)
- [ ] Gateway resource created (gatewayClassName: agentgateway)
- [ ] Service patched to NodePort
- [ ] LLM routing working
- [ ] MCP endpoint accessible

## Your Task
Work on the FIRST unchecked item. Complete it fully before moving on.

## Constraints
- Always validate: `kubectl apply --dry-run=client -f <file>`
- Test with curl before marking complete
- Use NodePort (LoadBalancer won't work in Kind)
- GatewayClass: agentgateway
- Namespace: agentgateway-system

## Docs
- https://kgateway.dev/docs/agentgateway/main/
EOF

# Start Ralph
while :; do cat PROMPT.md | claude ; done
```

---

## The PROMPT.md Template

Here's a comprehensive template for agentgateway work:

```markdown
# agentgateway POC

You are Ralph, an AI engineer building an agentgateway POC on Kind.

## 🎯 Mission
Build a working agentgateway deployment that can:
1. Route LLM traffic to cloud providers (OpenAI, Anthropic)
2. Expose MCP tool federation endpoint
3. Support agent-to-agent (A2A) connectivity

## 📍 Current State
<!-- Update this as you progress -->
- [x] Kind cluster running
- [x] Gateway API CRDs installed (v1.4.0)
- [x] agentgateway Helm chart deployed (v2.2.0-main)
- [ ] Gateway resource created
- [ ] Gateway service converted to NodePort
- [ ] LLM Backend configured (OpenAI)
- [ ] HTTPRoute for LLM traffic working
- [ ] MCP target configured
- [ ] MCP tools accessible
- [ ] End-to-end test passing

## 🔨 Your Task
Work on the FIRST unchecked item. Complete it fully before moving on.

## ⚠️ Constraints (READ CAREFULLY)

### Validation Rules
- ALWAYS run `kubectl apply --dry-run=client -f <file>` before applying
- ALWAYS test with curl after deploying
- NEVER mark a task complete without verification

### Kind-Specific Rules
- Use `type: NodePort` for services (LoadBalancer stays PENDING)
- Access pattern: `localhost:<hostPort>` via extraPortMappings
- Patch services: `kubectl patch svc <name> -n agentgateway-system -p '{"spec": {"type": "NodePort"}}'`

### agentgateway Specifics
- Helm charts: `oci://ghcr.io/kgateway-dev/charts/agentgateway`
- CRDs chart: `oci://ghcr.io/kgateway-dev/charts/agentgateway-crds`
- Namespace: `agentgateway-system`
- GatewayClass: `agentgateway`
- Version: v2.2.0-main

### Gateway API Rules
- Gateway and HTTPRoute are namespaced resources
- parentRefs in HTTPRoute must match Gateway name AND namespace
- Use `gateway.networking.k8s.io/v1` API version

### Error Handling
- If something fails 3 times, STOP
- Add the failure to SIGNS.md
- Try a completely different approach

## 🔗 Quick Commands

```bash
# Check agentgateway status
kubectl get pods -n agentgateway-system
kubectl get gateway -A
kubectl get httproute -A
kubectl get gatewayclass agentgateway

# Check services
kubectl get svc -n agentgateway-system

# Patch to NodePort
kubectl patch svc <gateway-name> -n agentgateway-system \
  -p '{"spec": {"type": "NodePort"}}'

# Get NodePort
kubectl get svc <name> -n agentgateway-system \
  -o jsonpath='{.spec.ports[0].nodePort}'

# Check logs
kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway

# Test LLM endpoint
curl http://localhost:30080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{"model": "gpt-4", "messages": [{"role": "user", "content": "Hi"}]}'
```

## 📝 Output Format
After each action, report:
1. What you did
2. The command/file created
3. Result (success/failure)
4. Next step
```

---

## Running Ralph

### Basic Loop

```bash
while :; do cat PROMPT.md | claude ; done
```

### With Logging

```bash
while :; do
  echo "=== Ralph Run: $(date) ===" | tee -a ralph.log
  cat PROMPT.md | claude 2>&1 | tee -a ralph.log
done
```

### With Manual Confirmation

```bash
while :; do
  cat PROMPT.md | claude
  read -p "Press Enter to continue or Ctrl+C to stop..."
done
```

---

## Tuning Ralph (Adding Signs)

When Ralph makes mistakes, add "signs" to the prompt to prevent repeating them.

### Example SIGNS.md

```markdown
# Signs for Ralph

### LoadBalancer doesn't work in Kind
**Problem**: Services stayed in `<pending>` state
**Fix**: Use `type: NodePort` instead

### Wrong GatewayClass
**Problem**: Used wrong gatewayClassName
**Fix**: Always use `agentgateway` (not `kgateway`)

### Missing API version
**Problem**: YAML rejected for missing apiVersion
**Fix**: Use `gateway.networking.k8s.io/v1`

### Helm chart URL
**Problem**: Used wrong OCI registry
**Fix**: Charts are at `oci://ghcr.io/kgateway-dev/charts/agentgateway`
```

### Common Signs to Add

```markdown
## 🚧 Signs (Learn From These)

### Networking
- LoadBalancer = PENDING in Kind → Use NodePort
- Get NodePort: `kubectl get svc <n> -o jsonpath='{.spec.ports[0].nodePort}'`

### Helm Charts
- agentgateway: `oci://ghcr.io/kgateway-dev/charts/agentgateway`
- agentgateway-crds: `oci://ghcr.io/kgateway-dev/charts/agentgateway-crds`
- Always use `--version v2.2.0-main`
- Always use `-n agentgateway-system`

### Gateway API
- CRDs: `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml`
- GatewayClass: `agentgateway`
- API version: `gateway.networking.k8s.io/v1`

### Debugging
- Pod logs: `kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway`
- Gateway status: `kubectl get gateway -A -o yaml`
- Events: `kubectl get events -n agentgateway-system --sort-by='.lastTimestamp'`
```

---

## Example Workflows

### Workflow 1: Create Gateway

```yaml
# manifests/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ai-gateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
    - name: http
      protocol: HTTP
      port: 8080
      allowedRoutes:
        namespaces:
          from: All
```

```bash
# Apply and verify
kubectl apply -f manifests/gateway.yaml
kubectl wait --for=condition=Programmed=True \
  -n agentgateway-system gateway/ai-gateway --timeout=60s

# Patch to NodePort
kubectl patch svc ai-gateway -n agentgateway-system \
  -p '{"spec": {"type": "NodePort"}}'

# Get access info
kubectl get svc ai-gateway -n agentgateway-system
```

### Workflow 2: Configure LLM Backend (OpenAI)

```yaml
# manifests/backends/openai.yaml
apiVersion: v1
kind: Secret
metadata:
  name: openai-api-key
  namespace: agentgateway-system
type: Opaque
stringData:
  api-key: "${OPENAI_API_KEY}"
---
apiVersion: gateway.kgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  llm:
    provider: openai
    apiKeySecretRef:
      name: openai-api-key
      key: api-key
```

### Workflow 3: Create HTTPRoute for LLM

```yaml
# manifests/httproute-llm.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: llm-route
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: ai-gateway
      namespace: agentgateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1/chat/completions
      backendRefs:
        - name: openai
          kind: AgentgatewayBackend
          group: gateway.kgateway.dev
```

### Workflow 4: Test LLM Routing

```bash
# Get the NodePort
NODE_PORT=$(kubectl get svc ai-gateway -n agentgateway-system \
  -o jsonpath='{.spec.ports[0].nodePort}')

# Test OpenAI routing
curl -X POST http://localhost:${NODE_PORT}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### Workflow 5: MCP Tool Server

```yaml
# manifests/backends/mcp-server.yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mcp-tools
  namespace: agentgateway-system
spec:
  mcp:
    target:
      host: mcp-server.default.svc.cluster.local
      port: 3000
```

---

## Troubleshooting

### GatewayClass Not Found

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

### Gateway Not Becoming Ready

```bash
# Check controller logs
kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway

# Check gateway status
kubectl describe gateway ai-gateway -n agentgateway-system
```

### Service External IP Pending

```bash
# Patch to NodePort (Kind doesn't support LoadBalancer)
kubectl patch svc ai-gateway -n agentgateway-system \
  -p '{"spec": {"type": "NodePort"}}'
```

### Connection Refused

```bash
# Verify pod is running
kubectl get pods -n agentgateway-system

# Check if service has endpoints
kubectl get endpoints -n agentgateway-system

# Verify NodePort mapping
kubectl get svc -n agentgateway-system -o wide
```

---

## References

### agentgateway Documentation
- Docs: https://kgateway.dev/docs/agentgateway/main/
- Quickstart: https://kgateway.dev/docs/agentgateway/main/quickstart/
- LLM Consumption: https://kgateway.dev/docs/agentgateway/main/llm/
- MCP Connectivity: https://kgateway.dev/docs/agentgateway/main/mcp/
- Agent Connectivity: https://kgateway.dev/docs/agentgateway/main/agent/
- Helm Reference: https://kgateway.dev/docs/agentgateway/main/reference/helm/

### Standalone agentgateway (non-Kubernetes)
- Docs: https://agentgateway.dev/docs/
- GitHub: https://github.com/agentgateway/agentgateway

### Gateway API
- Spec: https://gateway-api.sigs.k8s.io/
- CRDs v1.4.0: https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.4.0

### Ralph Method
- Original: https://ghuntley.com/ralph/

### Claude Code
- Docs: https://code.claude.com/docs/en/setup
- Install: `curl -fsSL https://claude.ai/install.sh | bash`

---

## Appendix: Kind Cluster Config

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: agentgateway-poc
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
      - containerPort: 30443
        hostPort: 30443
        protocol: TCP
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                 AGENTGATEWAY + RALPH QUICK REF                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  INSTALL CLAUDE CODE:                                            │
│    curl -fsSL https://claude.ai/install.sh | bash                │
│                                                                  │
│  START RALPH:                                                    │
│    while :; do cat PROMPT.md | claude ; done                     │
│                                                                  │
│  GATEWAY API CRDs:                                               │
│    kubectl apply -f https://github.com/kubernetes-sigs/          │
│      gateway-api/releases/download/v1.4.0/standard-install.yaml  │
│                                                                  │
│  AGENTGATEWAY HELM:                                              │
│    helm upgrade -i --create-namespace \                          │
│      -n agentgateway-system \                                    │
│      --version v2.2.0-main \                                     │
│      agentgateway-crds \                                         │
│      oci://ghcr.io/kgateway-dev/charts/agentgateway-crds         │
│                                                                  │
│    helm upgrade -i -n agentgateway-system \                      │
│      --version v2.2.0-main \                                     │
│      agentgateway \                                              │
│      oci://ghcr.io/kgateway-dev/charts/agentgateway              │
│                                                                  │
│  KEY INFO:                                                       │
│    Namespace:    agentgateway-system                             │
│    GatewayClass: agentgateway                                    │
│    Charts:       oci://ghcr.io/kgateway-dev/charts/agentgateway  │
│                                                                  │
│  DEBUGGING:                                                      │
│    kubectl get pods -n agentgateway-system                       │
│    kubectl get gateway -A                                        │
│    kubectl get httproute -A                                      │
│    kubectl logs -n agentgateway-system \                         │
│      -l app.kubernetes.io/name=agentgateway                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Version Info

| Component | Version |
|-----------|---------|
| Gateway API CRDs | v1.4.0 |
| agentgateway | v2.2.0-main |
| Kind | v0.25.0 |
| Claude Code | Native (latest) |

---

*Happy Ralphing! 🎉*
