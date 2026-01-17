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

- **kgateway**: Kubernetes-native API gateway built on Envoy and Gateway API (CNCF sandbox project, formerly Gloo)
- **agentgateway**: AI/ML traffic routing, MCP tool federation, A2A agent connectivity, and LLM consumption gateway
- **Kind cluster**: Local Kubernetes for rapid iteration

### Two Installation Paths

There are two ways to deploy agentgateway:

| Option | Namespace | Use Case |
|--------|-----------|----------|
| **kgateway + agentgateway** | `kgateway-system` | When you need both Envoy-based routing AND AI gateway |
| **Standalone agentgateway** | `agentgateway-system` | Pure AI gateway use cases (recommended for this POC) |

This guide covers **both** options.

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
# Kind (check https://kind.sigs.k8s.io for latest version)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-$(uname | tr '[:upper:]' '[:lower:]')-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Claude Code (RECOMMENDED: Native installation)
curl -fsSL https://claude.ai/install.sh | bash
# Reload shell
source ~/.bashrc  # or source ~/.zshrc for zsh

# Verify Claude Code
claude --version
claude doctor

# Alternative: npm installation (deprecated but works)
# npm install -g @anthropic-ai/claude-code

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# jq
sudo apt-get install jq  # or brew install jq on macOS
```

### API Keys

```bash
# For Claude Code authentication (if using API directly)
export ANTHROPIC_API_KEY="sk-ant-..."

# Optional: For testing LLM routing through agentgateway
export OPENAI_API_KEY="sk-..."
```

> **Note**: Claude Code can also authenticate via OAuth with your Claude.ai account (Pro/Max plan) or Anthropic Console.

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

### 2. Create Kind Cluster

```bash
# Create cluster config
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: agentgateway-poc
nodes:
  - role: control-plane
    extraPortMappings:
      # Map NodePort range to host
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

# Create the cluster
kind create cluster --config kind-config.yaml
```

### 3. Install Gateway API CRDs

```bash
# Install Gateway API v1.4.0 (current as of Jan 2025)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

### 4. Install agentgateway (Choose One Option)

**Option A: Standalone agentgateway (Recommended for AI-focused POC)**

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

**Option B: kgateway with agentgateway enabled**

```bash
# Install kgateway CRDs
helm upgrade -i --create-namespace \
  --namespace kgateway-system \
  --version v2.1.2 \
  kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds

# Install kgateway with agentgateway enabled
helm upgrade -i -n kgateway-system \
  kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
  --version v2.1.2 \
  --set agentgateway.enabled=true

# Verify installation
kubectl get pods -n kgateway-system
kubectl get gatewayclass kgateway
kubectl get gatewayclass agentgateway
```

### 5. Create the PROMPT.md

```bash
cat > PROMPT.md << 'EOF'
# kgateway + agentgateway POC

You are building a Proof of Concept for agentgateway on a Kind cluster.

## Current State
- [x] Kind cluster created
- [x] Gateway API CRDs installed (v1.4.0)
- [x] agentgateway installed via Helm
- [ ] Gateway resource created
- [ ] Basic HTTPRoute working
- [ ] LLM routing endpoint accessible

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
- GatewayClass name: agentgateway
- Namespace: agentgateway-system
EOF
```

### 6. Start Ralph

```bash
while :; do cat PROMPT.md | claude ; done
```

---

## The PROMPT.md Template

Here's a comprehensive template tailored for kgateway/agentgateway work:

```markdown
# kgateway + agentgateway POC

You are Ralph, an AI engineer building an agentgateway POC on Kind.

## 🎯 Mission
Build a working agentgateway deployment that can:
1. Route AI/LLM traffic to cloud providers (OpenAI, Anthropic)
2. Expose MCP tool federation endpoint
3. Support basic observability

## 📍 Current State
<!-- Update this as you progress -->
- [x] Kind cluster running
- [x] Gateway API CRDs installed (v1.4.0)
- [x] agentgateway Helm chart deployed (v2.2.0-main)
- [ ] Gateway resource created (gatewayClassName: agentgateway)
- [ ] Service converted to NodePort for Kind access
- [ ] HTTPRoute for /openai working
- [ ] HTTPRoute for /anthropic working
- [ ] MCP endpoint accessible
- [ ] End-to-end test passing

## 🔨 Your Task
Work on the FIRST unchecked item. Complete it fully before moving on.

## 📁 Project Structure
- `kind/cluster-config.yaml` - Kind cluster configuration
- `manifests/` - Kubernetes manifests
- `helm/values.yaml` - Helm values for agentgateway
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
- Use `type: NodePort` for services (LoadBalancer stays PENDING in Kind)
- Get node IP: `kubectl get nodes -o wide` (use InternalIP)
- For localhost access in Kind: `docker exec -it agentgateway-poc-control-plane ip addr`
- Access pattern: `localhost:<hostPort>` if using extraPortMappings, otherwise `<node-ip>:<nodeport>`

### Helm Rules
- Check existing values: `helm get values agentgateway -n agentgateway-system`
- Helm chart location: `oci://ghcr.io/kgateway-dev/charts/agentgateway`
- CRDs chart: `oci://ghcr.io/kgateway-dev/charts/agentgateway-crds`
- Current version: v2.2.0-main (dev) or v2.1.2 (stable)

### Gateway API Rules
- GatewayClass for agentgateway: `agentgateway`
- GatewayClass for kgateway (Envoy): `kgateway`
- Gateway and HTTPRoute are namespaced resources
- parentRefs in HTTPRoute must match Gateway name AND namespace

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

# Check agentgateway
kubectl get pods -n agentgateway-system
kubectl get gateway -A
kubectl get httproute -A
kubectl get gatewayclass

# Get service details
kubectl get svc -n agentgateway-system

# Convert to NodePort (if needed)
kubectl patch svc <gateway-name> -n agentgateway-system -p '{"spec": {"type": "NodePort"}}'

# Check logs
kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway

# Test endpoint (adjust port as needed)
curl -v http://localhost:30080/healthz
```

### Key Resources
- Gateway API: `gateway.networking.k8s.io/v1`
- agentgateway docs: https://kgateway.dev/docs/agentgateway/main/
- GatewayClass: agentgateway

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

### 2025-01-15: LoadBalancer doesn't work in Kind
**Problem**: Ralph kept creating LoadBalancer services that stayed in `<pending>`
**Sign Added**: "Use `type: NodePort` for services (LoadBalancer won't work in Kind)"

### 2025-01-15: Wrong GatewayClass name
**Problem**: Used `kgateway` instead of `agentgateway`
**Sign Added**: "GatewayClass for agentgateway is 'agentgateway', not 'kgateway'"

### 2025-01-15: Forgot Gateway API CRDs
**Problem**: Gateway resources failed with "no matches for kind Gateway"
**Sign Added**: Added CRD installation as explicit first step

### 2025-01-16: Wrong Helm chart URL
**Problem**: Used old cr.kgateway.dev instead of ghcr.io
**Sign Added**: "agentgateway charts are at oci://ghcr.io/kgateway-dev/charts/agentgateway"

### 2025-01-16: Helm upgrade without namespace
**Problem**: Created duplicate release in default namespace
**Sign Added**: "ALWAYS include `-n agentgateway-system` in helm commands"
```

### Common Signs to Add

Add these to your PROMPT.md as needed:

```markdown
## 🚧 Signs (Learn From These)

### Networking
- LoadBalancer = PENDING in Kind. Use NodePort.
- Get node IP: `kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'`
- MetalLB can provide LoadBalancer IPs in Kind if needed (but adds complexity)

### Helm Charts (IMPORTANT - URLs changed!)
- agentgateway charts: `oci://ghcr.io/kgateway-dev/charts/agentgateway`
- agentgateway-crds: `oci://ghcr.io/kgateway-dev/charts/agentgateway-crds`
- kgateway charts: `oci://cr.kgateway.dev/kgateway-dev/charts/kgateway`
- ALWAYS check existing values before upgrade: `helm get values <release> -n <namespace>`
- ALWAYS use the correct namespace flag

### Gateway API
- Install CRDs FIRST: `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml`
- GatewayClass for agentgateway: `agentgateway`
- GatewayClass for kgateway (Envoy): `kgateway`
- HTTPRoute parentRefs must match Gateway name AND namespace

### agentgateway Specifics
- Namespace: agentgateway-system (standalone) or kgateway-system (integrated)
- Enable in kgateway Helm: `agentgateway.enabled: true` (lowercase!)
- Note: `agentGateway` (camelCase) was the OLD spelling before v2.1

### Debugging
- Pod not starting? Check events: `kubectl describe pod <n> -n agentgateway-system`
- Service not accessible? Check endpoints: `kubectl get endpoints -n agentgateway-system`
- Route not working? Check Gateway status: `kubectl get gateway -A -o yaml`
- Check controller logs: `kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway`
```

---

## Example Workflows

### Workflow 1: Initial Setup with Standalone agentgateway

```markdown
## 🔨 Your Task
Set up the Kind cluster and install standalone agentgateway.

## Steps
1. Create Kind cluster with port mappings
2. Install Gateway API CRDs (v1.4.0)
3. Install agentgateway-crds Helm chart
4. Install agentgateway Helm chart
5. Verify pods are running
6. Verify GatewayClass exists

## Commands
```bash
# Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# agentgateway CRDs
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v2.2.0-main \
  agentgateway-crds oci://ghcr.io/kgateway-dev/charts/agentgateway-crds

# agentgateway
helm upgrade -i -n agentgateway-system \
  agentgateway oci://ghcr.io/kgateway-dev/charts/agentgateway \
  --version v2.2.0-main
```

## Verification
- [ ] `kubectl get nodes` shows Ready
- [ ] `kubectl get pods -n agentgateway-system` shows Running
- [ ] `kubectl get gatewayclass agentgateway` shows Accepted=True
```

### Workflow 2: Create Gateway and Test Route

```markdown
## 🔨 Your Task
Create a Gateway resource and test basic routing.

## Gateway Manifest
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ai-gateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway  # MUST be 'agentgateway'
  listeners:
    - name: http
      protocol: HTTP
      port: 8080
      allowedRoutes:
        namespaces:
          from: All
```

## Steps
1. Apply Gateway manifest
2. Wait for Gateway to be Programmed
3. Check the created service
4. Patch to NodePort if needed
5. Test with curl

## Commands
```bash
# Apply and wait
kubectl apply -f gateway.yaml
kubectl wait --for=condition=Programmed=True -n agentgateway-system gateway/ai-gateway --timeout=60s

# Check service
kubectl get svc -n agentgateway-system

# Patch to NodePort
kubectl patch svc ai-gateway -n agentgateway-system -p '{"spec": {"type": "NodePort"}}'

# Get NodePort
kubectl get svc ai-gateway -n agentgateway-system -o jsonpath='{.spec.ports[0].nodePort}'
```
```

### Workflow 3: Configure LLM Routing

```markdown
## 🔨 Your Task
Configure HTTPRoutes for LLM provider routing.

## Requirements
- Route /v1/chat/completions to OpenAI backend
- Route /v1/messages to Anthropic backend

## Steps
1. Create Backend resources (or ExternalName services)
2. Create HTTPRoutes with path matching
3. Test with curl

## Example HTTPRoute for OpenAI
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-route
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
        - name: openai-backend
          port: 443
```

## Verification
```bash
curl -X POST http://localhost:<nodeport>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${OPENAI_API_KEY}" \
  -d '{"model": "gpt-4", "messages": [{"role": "user", "content": "Hello"}]}'
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

### GatewayClass Not Found

**Symptom**: `no matches for kind "GatewayClass"`

**Fix**: Gateway API CRDs not installed
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

### Gateway Stuck in "Pending"

**Symptom**: Gateway never becomes Programmed

**Fix**: Check controller logs and events
```bash
kubectl describe gateway <n> -n agentgateway-system
kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway
```

### Service External IP is "pending"

**Symptom**: LoadBalancer service never gets an IP

**Fix**: Kind doesn't support LoadBalancer. Use NodePort:
```bash
kubectl patch svc <n> -n agentgateway-system -p '{"spec": {"type": "NodePort"}}'
```

---

## References

### kgateway/agentgateway Documentation
- Main docs: https://kgateway.dev/docs/
- agentgateway (main): https://kgateway.dev/docs/agentgateway/main/
- agentgateway (stable): https://kgateway.dev/docs/agentgateway/latest/
- Helm values reference: https://kgateway.dev/docs/agentgateway/main/reference/helm/
- GitHub: https://github.com/kgateway-dev/kgateway

### Gateway API
- Spec: https://gateway-api.sigs.k8s.io/
- CRD install (v1.4.0): https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.4.0

### Kind
- Quick start: https://kind.sigs.k8s.io/docs/user/quick-start/
- LoadBalancer workaround: https://kind.sigs.k8s.io/docs/user/loadbalancer/

### Ralph Method
- Original post: https://ghuntley.com/ralph/
- Key insight: "deterministically bad in an undeterministic world"

### Claude Code
- Setup docs: https://code.claude.com/docs/en/setup
- GitHub: https://github.com/anthropics/claude-code
- Install: `curl -fsSL https://claude.ai/install.sh | bash`

---

## Appendix: Complete Kind Cluster Config

```yaml
# kind/cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: agentgateway-poc
nodes:
  - role: control-plane
    extraPortMappings:
      # HTTP traffic
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
      # HTTPS traffic
      - containerPort: 30443
        hostPort: 30443
        protocol: TCP
      # Additional ports for testing
      - containerPort: 31000
        hostPort: 31000
        protocol: TCP
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
```

## Appendix: Helm Values for Standalone agentgateway

```yaml
# helm/agentgateway-values.yaml
controller:
  image:
    pullPolicy: Always  # For dev builds
  logLevel: debug       # Verbose logging for POC
  
# Service configuration for Kind
# Note: The Gateway resource creates its own service
# This is for the controller service if needed
```

## Appendix: Helm Values for kgateway with agentgateway

```yaml
# helm/kgateway-values.yaml
agentgateway:
  enabled: true       # Note: lowercase (changed from agentGateway in v2.1)

controller:
  image:
    pullPolicy: IfNotPresent

# For AI Gateway features with Envoy (deprecated in favor of agentgateway)
# gateway:
#   aiExtension:
#     enabled: true
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    RALPH METHOD QUICK REF                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  INSTALL CLAUDE CODE:                                            │
│    curl -fsSL https://claude.ai/install.sh | bash                │
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
│    PROMPT.md    - Ralph's brain (edit this to tune)              │
│    SIGNS.md     - Lessons learned                                │
│    CHANGELOG.md - What Ralph built                               │
│                                                                  │
│  HELM CHARTS:                                                    │
│    agentgateway: oci://ghcr.io/kgateway-dev/charts/agentgateway  │
│    kgateway:     oci://cr.kgateway.dev/kgateway-dev/charts/...   │
│                                                                  │
│  GATEWAY API CRDs (v1.4.0):                                      │
│    kubectl apply -f https://github.com/kubernetes-sigs/          │
│      gateway-api/releases/download/v1.4.0/standard-install.yaml  │
│                                                                  │
│  DEBUGGING:                                                      │
│    kubectl get pods -n agentgateway-system                       │
│    kubectl get gateway -A                                        │
│    kubectl get httproute -A                                      │
│    kubectl logs -n agentgateway-system -l app.kubernetes.io/...  │
│                                                                  │
│  TESTING:                                                        │
│    kubectl get svc -n agentgateway-system                        │
│    curl http://localhost:<nodeport>/healthz                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Version History

| Date | Version | Changes |
|------|---------|---------|
| 2025-01-17 | 1.0 | Initial guide with verified installation commands |

### Verified Against
- Gateway API: v1.4.0
- kgateway: v2.1.2 (stable), v2.2.0-main (dev)
- agentgateway standalone: v2.2.0-main
- Kind: v0.25.0
- Claude Code: v2.1.x (native installation)

---

*Happy Ralphing! 🎉*
