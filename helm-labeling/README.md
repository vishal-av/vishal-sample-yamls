# Helm Labeling Example

This example demonstrates how to use Helm command flags in Harness to apply labels at two levels:
1. **Helm Release Labels** - Labels stored in Helm release metadata
2. **Kubernetes Resource Labels** - Labels applied to actual K8s resources (pods, deployments, services)

## What This Demonstrates

### Option 1: Release-Level Labels
Using `--labels` flag to label the Helm release itself:
```bash
--labels team=platform,env=dev,owner=myorg
```

**Use Cases:**
- Track Helm releases by team/environment/owner
- Query releases: `helm list --selector team=platform`
- Audit and compliance at release level

### Option 2: Resource-Level Labels  
Using `--set commonLabels.*` to label Kubernetes resources:
```bash
--set commonLabels.team=platform --set commonLabels.env=dev
```

**Use Cases:**
- Cost allocation by label
- Resource monitoring and filtering
- Network policies and pod selection
- Query resources: `kubectl get pods -l team=platform`

## Files

| File | Description |
|------|-------------|
| `Chart.yaml` | Helm chart metadata |
| `values.yaml` | Default values (labels injected via command flags) |
| `templates/deployment.yaml` | Nginx deployment with label support |
| `templates/service.yaml` | ClusterIP service with label support |
| `service.yaml` | Harness service definition with command flags |
| `pipeline.yaml` | Harness pipeline with verification steps |

## Helm Chart Structure

```
helm-labeling/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Values (commonLabels defined here)
├── templates/
│   ├── deployment.yaml     # Deployment with label injection
│   └── service.yaml        # Service with label injection
├── service.yaml            # Harness service config
├── pipeline.yaml           # Harness pipeline
└── README.md              # This file
```

## How to Use in Harness

### Step 1: Create Service

1. Go to **Services** → **+ New Service**
2. Click **YAML** view
3. Paste contents of `service.yaml`
4. Update `connectorRef` to your GitHub connector
5. Ensure `folderPath: helm-labeling` points to this directory

### Step 2: Create Pipeline

1. Go to **Pipelines** → **+ Create Pipeline**
2. Click **YAML** view
3. Paste contents of `pipeline.yaml`
4. Save the pipeline

### Step 3: Run Pipeline

1. Select **Environment** and **Infrastructure**
2. Run the pipeline
3. Watch the verification steps show both types of labels working

## Verification Steps in Pipeline

The pipeline includes 3 verification steps:

### 1. Verify Helm Release Labels
Shows that the Helm release has labels and can be queried:
```bash
helm list --selector team=platform
```

### 2. Verify Kubernetes Resource Labels
Shows that K8s resources (pods, deployments, services) have labels:
```bash
kubectl get pods -l team=platform
kubectl get all -l owner=myorg
```

### 3. Label Usage Examples
Displays common query patterns for both label types

## Expected Output

After deployment, you should see:

**Helm Release Labels:**
```
$ helm list -n <namespace> --selector team=platform
NAME               NAMESPACE  STATUS    CHART           LABELS
nginx-with-labels  default    deployed  nginx-demo-1.0  team=platform,env=dev,owner=myorg
```

**Kubernetes Resource Labels:**
```
$ kubectl get pods -n <namespace> -l team=platform --show-labels
NAME                         READY   STATUS    LABELS
nginx-demo-xxx              1/1     Running   team=platform,env=dev,owner=myorg,cost-center=1234

$ kubectl get all -n <namespace> -l cost-center=1234
NAME                         READY   STATUS    AGE
pod/nginx-demo-xxx          1/1     Running   2m
deployment.apps/nginx-demo  2/2     2         2m
service/nginx-demo          ClusterIP        2m
```

## Command Flags Explained

In `service.yaml`, the `commandFlags` section demonstrates both approaches:

```yaml
commandFlags:
  # Release-level labels (stored in Helm release secret/configmap)
  - commandType: Install
    flag: "--labels team=platform,env=<+env.name>,owner=myorg"

  # Resource-level labels (applied to pods, deployments, services)
  - commandType: Install
    flag: "--set commonLabels.team=platform --set commonLabels.env=<+env.name> --set commonLabels.owner=myorg"
```

Both flags are applied during `helm install` and `helm upgrade`.

## Label Schema Used

| Label | Value | Purpose |
|-------|-------|---------|
| `team` | `platform` | Team ownership |
| `env` | `<+env.name>` | Environment (dynamic from Harness) |
| `owner` | `myorg` | Organization/customer |
| `cost-center` | `1234` | Cost allocation tracking |
| `pipeline` | `<+pipeline.name>` | Pipeline that deployed this (release labels only) |

## Use Cases Enabled

With this labeling approach, you can:

1. **Cost Allocation**
   ```bash
   kubectl get pods --all-namespaces -l cost-center=1234 -o json > cost-report.json
   ```

2. **Team Resource Tracking**
   ```bash
   kubectl get all --all-namespaces -l team=platform
   ```

3. **Environment Auditing**
   ```bash
   helm list --all-namespaces --selector env=prod
   ```

4. **Monitoring Integration**
   - Prometheus can scrape by label selector
   - Datadog can filter by labels
   - CloudWatch metrics by tag

5. **Network Policies**
   ```yaml
   podSelector:
     matchLabels:
       team: platform
   ```

## Deployment Strategy

This example uses **K8sRollingDeploy** (Rolling Deployment):
- Zero downtime
- Gradual rollout
- Automatic rollback on failure

## Docker Image

Uses public nginx image:
- **Image**: `nginx:1.25-alpine`
- **Size**: ~40MB
- **Purpose**: Lightweight demo

## Requirements

- Kubernetes cluster
- Harness Kubernetes connector
- GitHub connector (or use Harness File Store)
- Helm 3.x on delegate

## Testing Locally

You can test the Helm chart locally:

```bash
# Lint the chart
helm lint helm-labeling/

# Install with labels
helm install test-release helm-labeling/ \
  --set commonLabels.team=platform \
  --set commonLabels.env=dev \
  --labels team=platform,env=dev

# Verify release labels
helm list --selector team=platform

# Verify resource labels
kubectl get pods -l team=platform
```

## Troubleshooting

### Chart doesn't support commonLabels?

If your chart doesn't have `commonLabels` defined in `values.yaml`, add it:

```yaml
commonLabels: {}
```

And ensure templates reference it:

```yaml
metadata:
  labels:
    {{- with .Values.commonLabels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
```

### Helm release labels not queryable?

Ensure your delegate is using Helm 3.1+:
```bash
helm version
```

### Labels with special characters?

Use quotes in command flags:
```yaml
flag: '--set "podLabels.app\\.kubernetes\\.io/name=myapp"'
```

## Related Documentation

- [Harness Helm Deployments](https://developer.harness.io/docs/continuous-delivery/deploy-srv-diff-platforms/helm/helm-cd-quickstart/)
- [Kubernetes Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Helm Command Flags](https://helm.sh/docs/helm/helm_install/)

## Questions?

For issues or questions about this example, please open an issue in the repository.
