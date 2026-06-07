# Kubernetes Infrastructure GitOps

This repository contains GitOps configurations for deploying an OpenTelemetry (OTel) collection infrastructure across multiple Kubernetes environments (dev, test, prod) using ArgoCD.

## Overview

This project implements a scalable, multi-environment OpenTelemetry collector infrastructure with:

- **OTel Agent**: Runs as a DaemonSet on each node to collect metrics and logs from the host and pods
- **OTel Gateway**: Runs as a Deployment to aggregate and forward telemetry data to upstream backends

The infrastructure follows a tiered architecture where agents send data to local gateways, which then forward to upstream observability backends.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Kubernetes Cluster                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                                │
│  │   Node 1   │  │   Node 2   │  │   Node N   │                                │
│  │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────────┐ │                                │
│  │ │  OTel  │ │  │ │  OTel  │ │  │ │  OTel  │ │                                │
│  │ │ Agent  │ │  │ │ Agent  │ │  │ │ Agent  │ │                                │
│  │ │(DS)    │ │  │ │(DS)    │ │  │ │(DS)    │ │                                │
│  │ └────┬───┘ │  │ └────┬───┘ │  │ └────┬───┘ │                                │
│  └───────┼────┘  └───────┼────┘  └───────┼────┘                                │
│          │                │                │                                    │
│          └────────────────┴────────────────┘                                    │
│                          │                                                      │
│                  ┌───────┴───────┐                                              │
│                  │  OTel Gateway │                                              │
│                  │  (Deployment) │                                              │
│                  └───────┬───────┘                                              │
│                          │                                                      │
│                  ┌───────┴───────┐                                              │
│                  │  Upstream     │                                              │
│                  │  Backend      │                                              │
│                  │  (e.g.,       │                                              │
│                  │   Tempo,     │                                              │
│                  │   Jaeger,    │                                              │
│                  │   Prometheus)│                                              │
│                  └───────────────┘                                              │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Environment Topology

| Environment | Agent Mode | Gateway Replicas | Resources | Scale Method |
|-------------|-----------|------------------|-----------|--------------|
| **dev**     | DaemonSet | 1                | Minimal   | Manual       |
| **test**    | DaemonSet | 2                | Low       | Manual       |
| **prod**    | DaemonSet | 4-12 (HPA)       | High      | Auto/Manual  |

## Repository Structure

```
k8s-infrastructure-gitops/
├── apps/                           # Application configurations
│   └── opentelemetry/
│       ├── agent/                  # OTel Agent configurations
│       │   ├── base-agent.yaml     # Base DaemonSet config
│       │   ├── dev-values.yaml     # Dev environment overrides
│       │   ├── test-values.yaml    # Test environment overrides
│       │   └── prod-values.yaml    # Prod environment overrides (with HPA)
│       └── gateway/                # OTel Gateway configurations
│           ├── base-gateway.yaml   # Base Deployment config
│           ├── dev-values.yaml     # Dev environment overrides
│           ├── test-values.yaml    # Test environment overrides
│           └── prod-values.yaml    # Prod environment overrides (with HPA)
├── argocd-apps/                    # ArgoCD Application manifests
│   ├── dev/                        # Dev cluster apps
│   │   ├── otel-agent-app.yaml
│   │   └── otel-gateway-app.yaml
│   ├── test/                       # Test cluster apps
│   │   ├── otel-agent-app.yaml
│   │   └── otel-gateway-app.yaml
│   └── prod/                       # Prod cluster apps
│       ├── otel-agent-app.yaml
│       └── otel-gateway-app.yaml
└── bootstrap/                      # Bootstrap ArgoCD
    ├── root-bootstrap-dev.yaml
    ├── root-bootstrap-test.yaml
    └── root-bootstrap-prod.yaml
```

## Environments

### Development (dev)
- Local cluster deployment
- Minimal resource allocation
- Debug-level logging
- Single gateway replica

### Test (test)
- Separate test cluster
- Low resource allocation
- Info-level logging
- Two gateway replicas for basic HA

### Production (prod)
- Production cluster with HPA
- High resource allocation
- Optimized memory/batch processors
- Auto-scaling: 4-12 replicas based on CPU

## Deployment with ArgoCD

### Prerequisites

1. ArgoCD installed in your cluster
2. Gitea instance hosting this repository (configure URL in bootstrap files)
3. Kubernetes clusters for each environment

### Initial Bootstrap

1. **Install ArgoCD** in your cluster:

```bash
kubectl create namespace argocd
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. **Configure repository access** - Update the `repoURL` in each bootstrap file to point to your Git repository:

```yaml
# bootstrap/root-bootstrap-dev.yaml
spec:
  source:
    repoURL: https://github.com/your-org/k8s-infrastructure-gitops.git
```

3. **Apply bootstrap manifests** to deploy ArgoCD Applications:

```bash
# Deploy to dev cluster
kubectl apply -f bootstrap/root-bootstrap-dev.yaml

# Deploy to test cluster
kubectl apply -f bootstrap/root-bootstrap-test.yaml

# Deploy to prod cluster
kubectl apply -f bootstrap/root-bootstrap-prod.yaml
```

### Deploying to Different Clusters

The bootstrap ArgoCD Applications are configured with different destinations:

```yaml
# bootstrap/root-bootstrap-dev.yaml
spec:
  destination:
    server: https://kubernetes.default.svc  # Local cluster
    namespace: argocd

# For remote clusters, update the destination server:
spec:
  destination:
    server: https://prod-cluster.example.com  # Remote prod cluster
    namespace: argocd
```

#### Deploying with kubeconfig

For remote clusters, create a secret with the kubeconfig:

```bash
# Create secret for remote cluster access
kubectl create secret generic prod-cluster-kubeconfig \
  --from-file=cluster.kubeconfig=path/to/prod-config \
  -n argocd

# Then reference it in the Application
spec:
  destination:
    server: https://prod-cluster.example.com
    namespace: argocd
  syncOptions:
    - CreateNamespace=true
```

### ArgoCD Application Configuration

Each environment has two ArgoCD Applications:

#### otel-agent-app.yaml
- Deploys OTel Agent as DaemonSet
- Namespace: `monitoring-{env}`
- Uses base configuration + environment-specific values

#### otel-gateway-app.yaml
- Deploys OTel Gateway as Deployment
- Namespace: `monitoring-{env}`
- Aggregates data from agents and forwards to backend

### Syncing and Managing Applications

```bash
# List applications
argocd app list

# Sync an application
argocd app sync dev-otel-agent
argocd app sync dev-otel-gateway

# Sync all applications
argocd app sync dev-otel-agent dev-otel-gateway test-otel-agent test-otel-gateway prod-otel-agent prod-otel-gateway

# Force sync (ignore cache)
argocd app sync dev-otel-agent --force

# Rollback to previous state
argocd app rollback dev-otel-agent

# View sync status
argocd app get dev-otel-agent
```

## Scaling

### Horizontal Pod Autoscaling (Prod Only)

The production gateway has HPA configured:

```yaml
# apps/opentelemetry/gateway/prod-values.yaml
hpa:
  enabled: true
  minReplicas: 4
  maxReplicas: 12
  targetCPUUtilizationPercentage: 80
```

### Manual Scaling

#### Scale Gateway Deployment

```bash
# Scale dev gateway (1 replica)
kubectl scale deployment dev-otel-gateway-collector -n monitoring-dev --replicas=1

# Scale test gateway (2 replicas)
kubectl scale deployment test-otel-gateway-collector -n monitoring-test --replicas=2

# Scale prod gateway (4 replicas minimum)
kubectl scale deployment prod-otel-gateway-collector -n monitoring-prod --replicas=4
```

#### Scale Agent DaemonSet

Agent DaemonSets scale automatically based on node count. To force scale:

```bash
# This is not typically needed for DaemonSets
kubectl scale daemonset dev-otel-agent-collector -n monitoring-dev --replicas=<node-count>
```

### Vertical Scaling (Adjust Resources)

Edit the environment-specific values files:

```yaml
# apps/opentelemetry/gateway/prod-values.yaml
resources:
  limits:
    cpu: 2
    memory: 4Gi
  requests:
    cpu: 500m
    memory: 2Gi
```

## Configuration

### Base Configuration

The base configurations define the core behavior:

#### Agent (base-agent.yaml)
- Mode: DaemonSet
- Receives metrics/logs from nodes
- Sends to local gateway

#### Gateway (base-gateway.yaml)
- Mode: Deployment
- Receives OTLP (gRPC/HTTP)
- Forwards to upstream backend

### Environment-Specific Overrides

| Setting | dev | test | prod |
|---------|-----|------|------|
| Memory Limiter Check | 1s | 1s | 500ms |
| Memory Limit | 300Mi | 2Gi | 4Gi |
| Gateway Replicas | 1 | 2 | 4-12 (HPA) |
| Log Level | debug | info | info |

## Customizing Upstream Backend

Update the gateway exporter configuration:

```yaml
# apps/opentelemetry/gateway/base-gateway.yaml
config:
  exporters:
    otlp:
      endpoint: "your-observability-backend:4317"
      tls:
        insecure: true
```

Replace `your-observability-backend:4317` with your actual backend (Tempo, Jaeger, Prometheus, etc.).

## Troubleshooting

### Check ArgoCD Sync Status

```bash
argocd app get dev-otel-agent
argocd app get dev-otel-gateway
```

### View Agent Logs

```bash
kubectl logs -n monitoring-dev -l app.kubernetes.io/name=otel-agent-collector --tail=100 -f
```

### View Gateway Logs

```bash
kubectl logs -n monitoring-dev -l app.kubernetes.io/name=otel-gateway-collector --tail=100 -f
```

### Check HPA Status (prod)

```bash
kubectl get hpa -n monitoring-prod
kubectl describe hpa prod-otel-gateway-collector -n monitoring-prod
```

## Contributing

1. Create a feature branch
2. Make changes to the appropriate environment directory
3. Test in dev/test environments
4. Open a PR for prod changes
5. Changes auto-sync via ArgoCD

## License

MIT
