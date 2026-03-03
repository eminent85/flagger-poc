# Flagger Progressive Delivery POC on GKE

This repository demonstrates progressive delivery on Google Kubernetes Engine (GKE) using Flagger, FluxCD, and Istio. It includes two complete example applications showcasing both canary and blue/green deployment strategies.

## Overview

This POC provides a production-ready setup for:
- **GitOps**: Automated deployment using FluxCD v2
- **Service Mesh**: Istio for traffic management and observability
- **Progressive Delivery**: Flagger for automated canary and blue/green deployments
- **Metrics**: Prometheus for monitoring and metrics analysis
- **Demo Applications**: Two podinfo instances demonstrating different deployment strategies

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         GKE Cluster                          │
│                                                              │
│  ┌────────────────┐    ┌──────────────┐   ┌──────────────┐ │
│  │   FluxCD       │───▶│    Istio     │◀──│  Prometheus  │ │
│  │  (GitOps)      │    │ (Service Mesh│   │  (Metrics)   │ │
│  └────────────────┘    └──────────────┘   └──────────────┘ │
│           │                    │                   ▲         │
│           │                    │                   │         │
│           ▼                    ▼                   │         │
│  ┌────────────────┐    ┌──────────────┐          │         │
│  │   Flagger      │───▶│   podinfo    │──────────┘         │
│  │  (Progressive  │    │  - Canary    │                     │
│  │   Delivery)    │    │  - Blue/Green│                     │
│  └────────────────┘    └──────────────┘                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Repository Structure

```
flagger-poc/
├── README.md
├── clusters/
│   └── gke-demo/
│       ├── infrastructure.yaml       # Flux Kustomization for infrastructure
│       └── apps.yaml                 # Flux Kustomization for applications
│
├── infrastructure/
│   ├── sources/                      # Helm repository definitions
│   ├── istio/                        # Istio installation (base, istiod, gateway)
│   ├── prometheus/                   # Prometheus installation
│   ├── flagger/                      # Flagger installation
│   └── configs/                      # Istio Gateway configuration
│
└── apps/
    └── base/
        ├── podinfo-canary/           # Canary deployment example
        └── podinfo-bluegreen/        # Blue/green deployment example
```

## Prerequisites

Before you begin, ensure you have:

1. **GKE Cluster**: A running GKE cluster (Kubernetes 1.27+)
   ```bash
   # Example cluster creation
   gcloud container clusters create flagger-demo \
     --zone us-central1-a \
     --num-nodes 3 \
     --machine-type n1-standard-2
   ```

2. **kubectl**: Configured to access your GKE cluster
   ```bash
   gcloud container clusters get-credentials flagger-demo --zone us-central1-a
   ```

3. **Flux CLI**: Version 2.0 or later
   ```bash
   # Install Flux CLI
   curl -s https://fluxcd.io/install.sh | sudo bash

   # Verify installation
   flux --version
   ```

4. **GitHub Personal Access Token**: With repo permissions
   - Create at: https://github.com/settings/tokens
   - Scopes needed: `repo`

5. **Git**: For managing the repository
   ```bash
   git --version
   ```

## Quick Start

### 1. Fork this Repository

Fork this repository to your GitHub account so you can manage it via GitOps.

### 2. Clone Your Fork

```bash
git clone https://github.com/<your-username>/flagger-poc.git
cd flagger-poc
```

### 3. Bootstrap Flux

Export your environment variables:

```bash
export GITHUB_TOKEN=<your-github-token>
export GITHUB_USER=<your-github-username>
export GITHUB_REPO=flagger-poc
```

Bootstrap Flux to your GKE cluster:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=./clusters/gke-demo \
  --personal
```

This command will:
- Install Flux components in the `flux-system` namespace
- Create a deploy key in your GitHub repository
- Create and push a commit to your repository
- Start syncing your cluster state with the Git repository

### 4. Verify Installation

Check Flux installation:

```bash
# Verify Flux components
flux check

# Watch Flux reconciliation
flux get kustomizations --watch
```

Check infrastructure components:

```bash
# Verify Istio
kubectl get pods -n istio-system
kubectl get svc -n istio-system istio-ingressgateway

# Verify Prometheus
kubectl get pods -n monitoring

# Verify Flagger
kubectl get pods -n istio-system -l app.kubernetes.io/name=flagger

# Verify applications
kubectl get pods -n podinfo-canary
kubectl get pods -n podinfo-bluegreen

# Check Canary resources
kubectl get canary -A
```

All components should be in a healthy state before proceeding.

## Deployment Strategies

### Canary Deployment (podinfo-canary)

The canary deployment gradually shifts traffic from the stable version to the new version:

- **Traffic Progression**: 0% → 10% → 20% → 30% → 40% → 50% → 100%
- **Analysis Interval**: 1 minute between steps
- **Metrics Checked**:
  - Request success rate (minimum 99%)
  - Request duration P99 (maximum 500ms)
- **Rollback Threshold**: 5 failed checks

**Configuration**: `apps/base/podinfo-canary/canary.yaml:21`

### Blue/Green Deployment (podinfo-bluegreen)

The blue/green deployment runs analysis and then switches traffic instantly:

- **Traffic Progression**: 0% → (10 minutes of analysis) → 100%
- **Analysis Duration**: 10 iterations × 1 minute = 10 minutes
- **Metrics Checked**:
  - Request success rate (minimum 99%)
  - Request duration P99 (maximum 500ms)
- **Rollback Threshold**: 2 failed checks

**Configuration**: `apps/base/podinfo-bluegreen/canary.yaml:21`

## Triggering a Deployment

### Canary Deployment

1. Update the image tag in the deployment:
   ```bash
   # Edit the file
   vim apps/base/podinfo-canary/deployment.yaml

   # Change line 34 from:
   # image: ghcr.io/stefanprodan/podinfo:6.0.0
   # to:
   # image: ghcr.io/stefanprodan/podinfo:6.0.1
   ```

2. Commit and push:
   ```bash
   git add apps/base/podinfo-canary/deployment.yaml
   git commit -m "Update podinfo-canary to 6.0.1"
   git push
   ```

3. Watch the canary analysis:
   ```bash
   # Watch canary status
   watch kubectl get canary -n podinfo-canary

   # Detailed status
   kubectl describe canary podinfo -n podinfo-canary

   # Watch Flagger events
   kubectl get events -n podinfo-canary --watch
   ```

### Blue/Green Deployment

1. Update the image tag:
   ```bash
   vim apps/base/podinfo-bluegreen/deployment.yaml
   # Change image tag from 6.0.0 to 6.0.1
   ```

2. Commit and push:
   ```bash
   git add apps/base/podinfo-bluegreen/deployment.yaml
   git commit -m "Update podinfo-bluegreen to 6.0.1"
   git push
   ```

3. Watch the deployment:
   ```bash
   watch kubectl get canary -n podinfo-bluegreen
   kubectl describe canary podinfo -n podinfo-bluegreen
   ```

## Observing Progressive Delivery

### Get Istio Gateway IP

```bash
export GATEWAY_IP=$(kubectl -n istio-system get svc istio-ingressgateway \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Gateway IP: $GATEWAY_IP"
```

### Generate Traffic

Generate traffic to observe the canary deployment:

```bash
# Canary endpoint
while true; do curl -H "Host: podinfo-canary.example.com" http://$GATEWAY_IP; sleep 1; done

# Blue/green endpoint
while true; do curl -H "Host: podinfo-bluegreen.example.com" http://$GATEWAY_IP; sleep 1; done
```

### Watch Traffic Distribution

```bash
# View canary weight
kubectl -n podinfo-canary get canary podinfo -o jsonpath='{.status.canaryWeight}'

# View VirtualService traffic split
kubectl -n podinfo-canary get virtualservice podinfo -o yaml

# View all Flagger-managed resources
kubectl -n podinfo-canary get all
```

### Monitor Metrics in Prometheus

```bash
# Port-forward to Prometheus
kubectl -n monitoring port-forward svc/prometheus-server 9090:80

# Open browser to http://localhost:9090
# Example queries:
# - istio_requests_total{destination_workload="podinfo-primary"}
# - istio_request_duration_milliseconds_bucket{destination_workload="podinfo"}
```

### View Flagger Logs

```bash
kubectl -n istio-system logs -l app.kubernetes.io/name=flagger -f
```

## Demo Scenarios

### Scenario 1: Successful Canary Deployment

1. Update podinfo-canary from `6.0.0` to `6.0.1`
2. Flagger detects the change and starts analysis
3. Traffic gradually increases: 10% → 20% → 30% → 40% → 50%
4. All metrics pass thresholds
5. Primary version promoted to `6.0.1`
6. Canary version scaled down

**Expected Duration**: ~5-6 minutes

### Scenario 2: Failed Canary with Automatic Rollback

1. Update podinfo-canary with a bad configuration:
   ```yaml
   env:
   - name: PODINFO_UI_MESSAGE
     value: "error"
   # This causes the app to return errors
   ```
2. Flagger detects high error rate
3. Metrics fail threshold checks
4. After 5 failed checks, automatic rollback occurs
5. Traffic returns to 100% stable version

### Scenario 3: Blue/Green Instant Switch

1. Update podinfo-bluegreen from `6.0.0` to `6.0.1`
2. Flagger runs analysis for 10 minutes
3. All checks pass
4. Traffic switches instantly from blue (0%) to green (100%)
5. Blue version kept for rollback capability

**Expected Duration**: ~10 minutes

## Canary Status Reference

| Status | Description |
|--------|-------------|
| `Initialized` | Canary is ready and waiting for changes |
| `Progressing` | Canary analysis in progress, traffic shifting |
| `Promoting` | Analysis passed, promoting new version |
| `Finalising` | Cleaning up canary resources |
| `Succeeded` | Deployment completed successfully |
| `Failed` | Analysis failed, rollback triggered |

Check status:
```bash
kubectl get canary -A
```

## Troubleshooting

### Flux Not Reconciling

```bash
# Check Flux logs
flux logs --all-namespaces --follow

# Force reconciliation
flux reconcile kustomization infrastructure --with-source
flux reconcile kustomization apps --with-source
```

### Istio Sidecar Not Injected

```bash
# Check namespace labels
kubectl get namespace podinfo-canary -o yaml

# Should have: istio-injection=enabled
kubectl label namespace podinfo-canary istio-injection=enabled --overwrite

# Restart pods to inject sidecar
kubectl rollout restart deployment -n podinfo-canary
```

### Flagger Not Progressing Canary

```bash
# Check Flagger logs
kubectl -n istio-system logs -l app.kubernetes.io/name=flagger --tail=100

# Check canary status
kubectl -n podinfo-canary describe canary podinfo

# Check if Prometheus is reachable
kubectl -n istio-system exec -it deploy/flagger -- wget -O- http://prometheus-server.monitoring:80/api/v1/query?query=up
```

### HelmRelease Failed

```bash
# List all HelmReleases
flux get helmreleases -A

# Check specific release
kubectl -n istio-system describe helmrelease istiod

# View Helm controller logs
kubectl -n flux-system logs -l app=helm-controller --tail=50
```

### No Metrics in Prometheus

```bash
# Check Prometheus targets
kubectl -n monitoring port-forward svc/prometheus-server 9090:80
# Visit http://localhost:9090/targets

# Check if Istio is exposing metrics
kubectl -n podinfo-canary exec -it deploy/podinfo-primary -c istio-proxy -- curl localhost:15090/stats/prometheus
```

### Gateway Not Getting External IP

```bash
# Check LoadBalancer service
kubectl -n istio-system get svc istio-ingressgateway

# Describe service for events
kubectl -n istio-system describe svc istio-ingressgateway

# On GKE, ensure your project has available external IPs
```

## Advanced Configuration

### Enable mTLS

Edit `infrastructure/istio/istiod.yaml`:

```yaml
values:
  meshConfig:
    enableAutoMtls: true
```

Update canary resources to use `ISTIO_MUTUAL` mode.

### Add Custom Metrics

Edit canary resource to add custom Prometheus queries:

```yaml
analysis:
  metrics:
  - name: custom-metric
    thresholdRange:
      min: 90
    query: |
      your_custom_prometheus_query
```

### Configure Slack Notifications

Add to `infrastructure/flagger/release.yaml`:

```yaml
values:
  slack:
    url: https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
    channel: flagger-notifications
    user: flagger
```

### Add Load Testing

Flagger can run automated load tests during canary analysis. The webhook references in the canary resources are configured for the flagger-loadtester tool.

To install flagger-loadtester:

```bash
kubectl create namespace test
kubectl apply -k github.com/fluxcd/flagger//kustomize/tester
```

## Cleanup

To remove all components from your cluster:

```bash
# Uninstall Flux
flux uninstall --silent

# Delete namespaces
kubectl delete namespace istio-system monitoring podinfo-canary podinfo-bluegreen

# Delete CRDs (optional)
kubectl delete crd $(kubectl get crd | grep istio.io | awk '{print $1}')
```

To delete your GKE cluster:

```bash
gcloud container clusters delete flagger-demo --zone us-central1-a
```

## Resources

- [Flagger Documentation](https://docs.flagger.app)
- [FluxCD Documentation](https://fluxcd.io/flux/)
- [Istio Documentation](https://istio.io/latest/docs/)
- [Podinfo Application](https://github.com/stefanprodan/podinfo)

## Next Steps

1. **Add A/B Testing**: Configure Flagger for A/B testing with header-based routing
2. **Implement Alerting**: Set up Slack or Teams notifications for deployment events
3. **Add Custom Metrics**: Define application-specific metrics for analysis
4. **Configure Load Testing**: Use flagger-loadtester for automated testing
5. **Add Grafana Dashboards**: Visualize Flagger metrics and deployment progress
6. **Multi-Environment Setup**: Extend to staging and production environments

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
