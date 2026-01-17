# Ralph Method: kgateway + agentgateway POC Guide

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

This guide walks you through using the **Ralph Method** with **Claude Code** to build and test a kgateway + agentgateway Proof of Concept on a local Kind cluster.

### What We're Building

- **kgateway**: Kubernetes-native API gateway built on Envoy and Gateway API
- **agentgateway**: AI/ML traffic routing, MCP tool federation, and agent connectivity
- **Kind cluster**: Local Kubernetes for rapid iteration

### Why Ralph Works for This

| Ralph Strength | kgateway/agentgateway Fit |
|----------------|---------------------------|
| Greenfield iteration | New Gateway/HTTPRoute configs iterate cleanly |
| Predictable failures | YAML errors, CRD mismatches are learnable |
| Self-testing loop | `kubectl apply`, `curl`, `helm test` provide feedback |
| Eventual consistency | Complex multi-resource deployments converge |

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
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-$(uname)-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Claude Code (requires Node.js 18+)
npm install -g @anthropic-ai/claude-code

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# jq
sudo apt-get install jq  # or brew install jq
```

### API Keys

```bash
# Set your Anthropic API key for Claude Code
export ANTHROPIC_API_KEY="sk-ant-..."

# Optional: OpenAI key for testing agentgateway routing
export OPENAI_API_KEY="sk-..."
```

---

## Project Structure

```
kgateway-poc/
├── PROMPT.md              # Ralph's instructions (the brain)
├── CHANGELOG.md           # Track what Ralph builds
├── SIGNS.md               # Learnings from Ralph's mistakes
├── kind/
│   └── cluster-config.yaml
├── manifests/
│   ├── gateway.yaml
│   ├── httproute.yaml
│   └── backends/
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
mkdir -p kgateway-poc && cd kgateway-poc
```

### 2. Create the PROMPT.md

```bash
cat > PROMPT.md << 'EOF'
# kgateway + agentgateway POC

You are building a Proof of Concept for kgateway with agentgateway on a Kind cluster.

## Current State
- [ ] Kind cluster created
- [ ] kgateway installed via Helm
- [ ] agentgateway enabled
- [ ] Basic HTTPRoute working
- [ ] MCP endpoint accessible

## Your Task
Check the current state, then work on the next uncompleted item.
Update this checklist as you complete tasks.

## Constraints
- Always validate manifests: `kubectl apply --dry-run=client -f <file>`
- Test endpoints with curl before marking complete
- Update CHANGELOG.md after each feature
- If something fails 3 times, add a note to SIGNS.md and try a different approach

## Context
- kgateway docs: https://kgateway.dev/docs/
- agentgateway docs: https://kgateway.dev/docs/agentgateway/main/
- Use NodePort (not LoadBalancer) for Kind compatibility
EOF
```

### 3. Start Ralph

```bash
while :; do cat PROMPT.md | claude ; done
```

---

## The PROMPT.md Template

Here's a comprehensive template tailored for kgateway/agentgateway work:

```markdown
# kgateway + agentgateway POC

You are Ralph, an AI engineer building a kgateway + agentgateway POC on Kind.

## 🎯 Mission
Build a working agentgateway deployment that can:
1. Route AI/LLM traffic to multiple backends
2. Expose MCP tool federation endpoint
3. Support basic observability

## 📍 Current State
<!-- Update this as you progress -->
- [x] Kind cluster running
- [ ] Gateway API CRDs installed
- [ ] kgateway Helm chart deployed
- [ ] agentgateway enabled
- [ ] Gateway resource created
- [ ] HTTPRoute for /openai working
- [ ] HTTPRoute for /anthropic working
- [ ] MCP endpoint accessible
- [ ] Token-based rate limiting configured
- [ ] End-to-end test passing

## 🔨 Your Task
Work on the FIRST unchecked item. Complete it fully before moving on.

## 📁 Project Structure
- `kind/cluster-config.yaml` - Kind cluster configuration
- `manifests/` - Kubernetes manifests
- `helm/values.yaml` - Helm values for kgateway
- `scripts/` - Setup, test, and teardown scripts
- `tests/` - Test scripts and curl commands
- `CHANGELOG.md` - What you've built
- `SIGNS.md` - Lessons learned from failures

## ⚠️ Constraints (READ CAREFULLY)

### Validation Rules
- ALWAYS run `kubectl apply --dry-run=client -f <file>` before applying
- ALWAYS test with curl after deploying
- NEVER mark a task complete without verification

### Kind-Specific Rules
- Use `type: NodePort` for services (LoadBalancer won't work)
- Get node IP: `kubectl get nodes -o wide`
- Access pattern: `<node-ip>:<nodeport>`

### Helm Rules
- Check existing values: `helm get values kgateway -n kgateway-system`
- Upgrade command: `helm upgrade kgateway kgateway/kgateway -n kgateway-system -f helm/values.yaml`

### File Management
- Create missing directories before writing files
- Use relative paths in manifests

### Error Handling
- If something fails 3 times, STOP
- Add the failure to SIGNS.md
- Try a completely different approach
- Ask for help if stuck

## 🔗 Quick References

### Useful Commands
```bash
# Check cluster
kubectl cluster-info
kubectl get nodes -o wide

# Check kgateway
kubectl get pods -n kgateway-system
kubectl get gateway -A
kubectl get httproute -A

# Check agentgateway
kubectl get svc agentgateway -n kgateway-system
kubectl logs -n kgateway-system -l app=agentgateway

# Test endpoint
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
NODE_PORT=$(kubectl get svc agentgateway -n kgateway-system -o jsonpath='{.spec.ports[0].nodePort}')
curl -v http://${NODE_IP}:${NODE_PORT}/healthz
```

### Key Resources
- Gateway API: `gateway.networking.k8s.io/v1`
- kgateway docs: https://kgateway.dev/docs/
- agentgateway: https://kgateway.dev/docs/agentgateway/main/

## 📝 Output Format
After each action, report:
1. What you did
2. The command/file you created
3. The result (success/failure)
4. Next step
```

---

## Running Ralph

### Basic Loop

```bash
# Simple infinite loop
while :; do cat PROMPT.md | claude ; done
```

### With Logging

```bash
# Log all output with timestamps
while :; do
  echo "=== Ralph Run: $(date) ===" | tee -a ralph.log
  cat PROMPT.md | claude 2>&1 | tee -a ralph.log
  echo "" | tee -a ralph.log
done
```

### With Pause Between Runs

```bash
# Add a 5-second pause between runs
while :; do
  cat PROMPT.md | claude
  echo "Sleeping 5s before next run..."
  sleep 5
done
```

### With Manual Confirmation

```bash
# Pause for review between runs
while :; do
  cat PROMPT.md | claude
  echo ""
  read -p "Press Enter to continue or Ctrl+C to stop..."
done
```

### With Cost Tracking

```bash
# Track API costs (approximate)
RUN_COUNT=0
while :; do
  RUN_COUNT=$((RUN_COUNT + 1))
  echo "=== Run #${RUN_COUNT} ==="
  cat PROMPT.md | claude
done
```

---

## Tuning Ralph (Adding Signs)

When Ralph makes mistakes, you "tune" Ralph by adding guidance to the prompt. Think of these as signs at a playground telling Ralph not to jump off the slide.

### Create SIGNS.md

```markdown
# Signs for Ralph

## Lessons Learned

### 2024-01-15: LoadBalancer doesn't work in Kind
**Problem**: Ralph kept creating LoadBalancer services that stayed in `<pending>`
**Sign Added**: "Use `type: NodePort` for services (LoadBalancer won't work in Kind)"

### 2024-01-15: Forgot to install Gateway API CRDs
**Problem**: Gateway resources failed with "no matches for kind Gateway"
**Sign Added**: Added CRD installation as explicit first step in checklist

### 2024-01-16: Wrong agentgateway port
**Problem**: Kept curling port 8080 when it was on 8081
**Sign Added**: "Always check actual port: `kubectl get svc -n kgateway-system`"

### 2024-01-16: Helm upgrade without namespace
**Problem**: Created duplicate release in default namespace
**Sign Added**: "ALWAYS include `-n kgateway-system` in helm commands"
```

### Common Signs to Add

Add these to your PROMPT.md as needed:

```markdown
## 🚧 Signs (Learn From These)

### Networking
- LoadBalancer = PENDING in Kind. Use NodePort.
- Get node IP: `kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'`
- MetalLB can provide LoadBalancer IPs in Kind if needed

### Helm
- ALWAYS check existing values before upgrade: `helm get values kgateway -n kgateway-system`
- ALWAYS use `-n kgateway-system` flag
- Use `--reuse-values` to preserve existing config

### Gateway API
- Install CRDs FIRST: `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml`
- Gateway must reference correct GatewayClass: `kgateway`
- HTTPRoute parentRefs must match Gateway name AND namespace

### agentgateway
- Enable in Helm values: `agentgateway.enabled: true`
- Check logs: `kubectl logs -n kgateway-system -l app.kubernetes.io/name=agentgateway`
- Default port is 8080

### Debugging
- Pod not starting? Check events: `kubectl describe pod <name> -n kgateway-system`
- Service not accessible? Check endpoints: `kubectl get endpoints -n kgateway-system`
- Route not working? Check Gateway status: `kubectl get gateway -n kgateway-system -o yaml`
```

---

## Example Workflows

### Workflow 1: Initial Setup

```markdown
## 🔨 Your Task
Set up the Kind cluster and install kgateway.

## Steps
1. Create Kind cluster with port mappings
2. Install Gateway API CRDs
3. Add kgateway Helm repo
4. Install kgateway with agentgateway enabled
5. Verify pods are running

## Verification
- [ ] `kubectl get nodes` shows Ready
- [ ] `kubectl get pods -n kgateway-system` shows Running
- [ ] `kubectl get gateway -n kgateway-system` shows Accepted
```

### Workflow 2: Configure AI Routing

```markdown
## 🔨 Your Task
Configure HTTPRoutes for AI backend routing.

## Requirements
- Route /v1/chat/completions to OpenAI
- Route /v1/messages to Anthropic
- Apply token-based rate limiting

## Steps
1. Create ExternalName services for backends
2. Create HTTPRoutes with path matching
3. Test with curl

## Verification
```bash
curl -X POST http://${GATEWAY_URL}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${OPENAI_API_KEY}" \
  -d '{"model": "gpt-4", "messages": [{"role": "user", "content": "Hello"}]}'
```
```

### Workflow 3: MCP Tool Federation

```markdown
## 🔨 Your Task
Set up MCP tool federation endpoint.

## Requirements
- Expose MCP SSE endpoint
- Configure tool routing
- Test with MCP client

## Steps
1. Enable MCP in agentgateway config
2. Create MCP-specific HTTPRoute
3. Configure backend MCP servers
4. Test SSE connection

## Verification
```bash
curl -N http://${GATEWAY_URL}/mcp/sse \
  -H "Accept: text/event-stream"
```
```

---

## Troubleshooting

### Ralph Gets Stuck in a Loop

**Symptom**: Same error repeated, no progress

**Fix**: Add a circuit breaker to your prompt:
```markdown
## ⛔ Circuit Breaker
If you see the same error 3 times:
1. STOP trying that approach
2. Document the failure in SIGNS.md
3. Try a completely different method
4. If still stuck, output "RALPH NEEDS HELP: <description>"
```

### Ralph Creates Invalid YAML

**Symptom**: `kubectl apply` fails with syntax errors

**Fix**: Add validation step:
```markdown
## Validation (REQUIRED)
Before creating any YAML file:
1. Create the file
2. Run: `kubectl apply --dry-run=client -f <file>`
3. If it fails, fix and retry
4. Only proceed when dry-run passes
```

### Ralph Forgets Previous Context

**Symptom**: Recreates files that already exist

**Fix**: Add state checking:
```markdown
## Before Starting
1. Check what exists: `ls -la manifests/`
2. Check cluster state: `kubectl get all -n kgateway-system`
3. Read CHANGELOG.md for history
4. Only work on what's NOT done
```

### Ralph Uses Wrong Commands

**Symptom**: Commands fail because of wrong syntax

**Fix**: Add command reference:
```markdown
## Command Cheatsheet (USE THESE EXACT COMMANDS)
- Install CRDs: `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml`
- Helm install: `helm install kgateway kgateway/kgateway -n kgateway-system --create-namespace`
- Helm upgrade: `helm upgrade kgateway kgateway/kgateway -n kgateway-system -f helm/values.yaml`
```

---

## References

### kgateway Documentation
- Main docs: https://kgateway.dev/docs/
- agentgateway: https://kgateway.dev/docs/agentgateway/main/
- Helm chart: https://github.com/k8sgateway/k8sgateway

### Gateway API
- Spec: https://gateway-api.sigs.k8s.io/
- CRD install: https://gateway-api.sigs.k8s.io/guides/#installing-gateway-api

### Kind
- Quick start: https://kind.sigs.k8s.io/docs/user/quick-start/
- Ingress: https://kind.sigs.k8s.io/docs/user/ingress/

### Ralph Method
- Original post: https://ghuntley.com/ralph/
- Key insight: "deterministically bad in an undeterministic world"

### Claude Code
- Docs: https://docs.anthropic.com/en/docs/claude-code
- GitHub: https://github.com/anthropics/claude-code

---

## Appendix: Complete Kind Cluster Config

```yaml
# kind/cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: kgateway-poc
nodes:
  - role: control-plane
    extraPortMappings:
      # agentgateway NodePort range
      - containerPort: 30000
        hostPort: 30000
        protocol: TCP
      - containerPort: 30001
        hostPort: 30001
        protocol: TCP
      - containerPort: 32000
        hostPort: 32000
        protocol: TCP
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
```

## Appendix: Helm Values for agentgateway

```yaml
# helm/values.yaml
agentgateway:
  enabled: true
  
controller:
  image:
    tag: latest
    
gateway:
  enabled: true
  
# For Kind compatibility
service:
  type: NodePort
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    RALPH METHOD QUICK REF                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  START RALPH:                                                    │
│    while :; do cat PROMPT.md | claude ; done                     │
│                                                                  │
│  STOP RALPH:                                                     │
│    Ctrl+C                                                        │
│                                                                  │
│  TUNE RALPH:                                                     │
│    1. Identify repeated failure                                  │
│    2. Add "sign" to PROMPT.md                                    │
│    3. Restart loop                                               │
│                                                                  │
│  KEY FILES:                                                      │
│    PROMPT.md   - Ralph's brain (edit this to tune)               │
│    SIGNS.md    - Lessons learned                                 │
│    CHANGELOG.md - What Ralph built                               │
│                                                                  │
│  DEBUGGING:                                                      │
│    kubectl get pods -n kgateway-system                           │
│    kubectl logs -n kgateway-system -l app.kubernetes.io/name=... │
│    kubectl describe gateway -n kgateway-system                   │
│                                                                  │
│  TESTING:                                                        │
│    NODE_IP=$(kubectl get nodes -o jsonpath='{...}')              │
│    NODE_PORT=$(kubectl get svc ... -o jsonpath='{...}')          │
│    curl http://${NODE_IP}:${NODE_PORT}/healthz                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

*Happy Ralphing! 🎉*
