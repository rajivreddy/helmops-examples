# HelmOps Helm Chart

A Helm chart for deploying [HelmOp](https://fleet.rancher.io/) resources to Rancher Fleet.

## Overview

This chart creates HelmOp Custom Resources that enable Fleet to deploy Helm charts to multiple downstream Kubernetes clusters. HelmOp provides a Kubernetes-native way to manage Helm releases across your fleet of clusters.

**Key Feature:** This chart supports deploying **multiple HelmOp resources** from a single values file, allowing you to manage multiple applications across clusters with one Helm release.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- Rancher with Fleet installed (v2.5+)
- Target clusters registered with Rancher Fleet

## Installation

### Add the Helm Repository

```bash
helm repo add helmops-examples https://rajivreddy.github.io/helmops-examples/
helm repo update
```

### Install the Chart

```bash
# Using default values
helm install my-helmop helmops-examples/helmops

# Using custom values file
helm install my-helmop helmops-examples/helmops -f my-values.yaml

# Deploy multiple HelmOps
helm install multi-app helmops-examples/helmops -f multi-helmops-values.yaml
```

## Configuration

### Multi-HelmOps Mode (Recommended)

Deploy multiple HelmOp resources from a single values file using the `helmops` list:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.helmopNamespace` | Default namespace for all HelmOp resources | `fleet-local` |
| `global.labels` | Labels applied to all HelmOp resources | `{}` |
| `global.annotations` | Annotations applied to all HelmOp resources | `{}` |
| `helmops` | List of HelmOp configurations | `[]` |

Each item in `helmops` list:

| Field | Description | Required |
|-------|-------------|----------|
| `name` | HelmOp resource name | Yes |
| `targetNamespace` | Namespace for Helm release on target clusters | No (`default`) |
| `helmopNamespace` | Override global helmopNamespace | No |
| `helm.releaseName` | Helm release name | No (defaults to `name`) |
| `helm.chart` | Chart name in repository | Yes |
| `helm.repo` | Helm repository URL | Yes |
| `helm.version` | Chart version | No |
| `helm.values` | Default values for all targets | No |
| `targets` | List of target cluster configurations | Yes |
| `labels` | Per-HelmOp labels | No |
| `annotations` | Per-HelmOp annotations | No |

### Single HelmOp Mode (Backward Compatible)

For deploying a single HelmOp, use the root-level parameters:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `name` | HelmOp resource name | `""` |
| `helmopNamespace` | Namespace for HelmOp resource | `fleet-local` |
| `targetNamespace` | Namespace where Helm release deploys | `default` |
| `helm.releaseName` | Helm release name | `""` |
| `helm.chart` | Chart name in the repository | `""` |
| `helm.repo` | Helm repository URL | `""` |
| `helm.version` | Chart version (optional) | `""` |
| `helm.values` | Default values applied to all targets | `{}` |
| `helm.valuesFiles` | List of values files to use | `[]` |
| `targets` | List of target cluster configurations | `[]` |
| `labels` | Additional labels for HelmOp resource | `{}` |
| `annotations` | Additional annotations for HelmOp resource | `{}` |

### Target Configuration

Each target in the `targets` list supports:

| Field | Description | Required |
|-------|-------------|----------|
| `name` | Unique identifier for the target | Yes |
| `clusterName` | Name of the registered cluster in Fleet | Yes |
| `helm.values` | Per-cluster value overrides | No |
| `helm.valuesFiles` | Per-cluster values files | No |

## Examples

### Multiple HelmOps (Recommended)

Deploy nginx, redis, and prometheus with a single Helm release:

```yaml
# multi-helmops-values.yaml
global:
  helmopNamespace: fleet-local
  labels:
    managed-by: helmops-chart

helmops:
  # nginx deployment
  - name: nginx
    targetNamespace: default
    helm:
      releaseName: nginx
      chart: nginx
      repo: https://rajivreddy.github.io/helmops-examples/
    targets:
      - name: k3s
        clusterName: "k3s"
        helm:
          values:
            replicaCount: 1
      - name: prom-agent
        clusterName: "prom-agent"
        helm:
          values:
            replicaCount: 2

  # redis deployment
  - name: redis
    targetNamespace: redis-system
    helm:
      releaseName: redis
      chart: redis
      repo: https://charts.bitnami.com/bitnami
      version: "17.0.0"
    targets:
      - name: k3s
        clusterName: "k3s"
        helm:
          values:
            architecture: standalone

  # prometheus deployment
  - name: prometheus
    targetNamespace: monitoring
    helm:
      releaseName: prometheus
      chart: prometheus
      repo: https://prometheus-community.github.io/helm-charts
    targets:
      - name: prom-agent
        clusterName: "prom-agent"
```

```bash
helm install multi-app charts/helmops -f multi-helmops-values.yaml
```

### Single HelmOp (Basic Usage)

```yaml
# values.yaml
name: my-app

helm:
  releaseName: my-app
  chart: nginx
  repo: https://charts.bitnami.com/bitnami

targets:
  - name: dev-cluster
    clusterName: "dev"
```

### Multi-Cluster Deployment

```yaml
# values.yaml
name: nginx

helmopNamespace: fleet-local
targetNamespace: default

helm:
  releaseName: nginx
  chart: nginx
  repo: https://rajivreddy.github.io/helmops-examples/

targets:
  - name: k3s
    clusterName: "k3s"
    helm:
      values:
        replicaCount: 1

  - name: prom-agent
    clusterName: "prom-agent"
    helm:
      values:
        image:
          tag: "alpine3.23"
        replicaCount: 2
        service:
          type: NodePort
```

### With Default Values for All Targets

```yaml
# values.yaml
name: nginx

helm:
  releaseName: nginx
  chart: nginx
  repo: https://rajivreddy.github.io/helmops-examples/
  values:
    image:
      repository: nginx
      tag: stable
    service:
      type: ClusterIP

targets:
  - name: dev
    clusterName: "dev-cluster"
    helm:
      values:
        replicaCount: 1

  - name: prod
    clusterName: "prod-cluster"
    helm:
      values:
        replicaCount: 3
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
```

### With Chart Version

```yaml
# values.yaml
name: nginx

helm:
  releaseName: nginx
  chart: nginx
  repo: https://rajivreddy.github.io/helmops-examples/
  version: "0.1.0"

targets:
  - name: production
    clusterName: "production"
```

## Verifying Installation

After installing the chart, verify the HelmOp resources:

```bash
# Check all HelmOps status
kubectl get helmops -n fleet-local

# Describe for details
kubectl describe helmop <name> -n fleet-local

# Check Fleet bundles created
kubectl get bundles -n fleet-default
```

## Troubleshooting

### HelmOp Not Creating Bundles

1. Verify the Helm repository URL is accessible
2. Check that the chart name exists in the repository
3. Review Fleet controller logs:
   ```bash
   kubectl logs -n cattle-fleet-system -l app=fleet-controller
   ```

### Cluster Not Receiving Deployment

1. Ensure `clusterName` matches exactly with the registered cluster name in Fleet
2. Verify the cluster is connected:
   ```bash
   kubectl get clusters.fleet.cattle.io -n fleet-default
   ```

### Values Not Applied

1. Check YAML syntax in `helm.values` section
2. Verify indentation is correct
3. Use `helm template` to preview the generated manifest:
   ```bash
   helm template my-release charts/helmops -f values.yaml
   ```

## Uninstalling

```bash
helm uninstall my-helmop
```

This removes the HelmOp resource, which triggers Fleet to remove the Helm release from target clusters.

## License

This chart is provided as an example for educational purposes.
