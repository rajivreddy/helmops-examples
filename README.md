# Helm Ops Examples - Rancher Fleet Deployment

This repository contains Helm chart examples configured for deployment using **Rancher Fleet HelmOp** CRD.

## Overview

Rancher Fleet is a GitOps at scale solution for managing Kubernetes clusters. This project demonstrates how to deploy Helm charts using Fleet's **HelmOp** Custom Resource Definition (CRD), which provides a Kubernetes-native way to manage Helm releases across multiple clusters.

## Prerequisites

- Rancher with Fleet installed (v2.5+)
- One or more downstream Kubernetes clusters registered with Rancher
- Helm chart hosted in a Helm repository (e.g., GitHub Pages)
- `kubectl` configured to access your Fleet management cluster
- `helm` CLI (optional, for local testing)

## Repository Structure

```
helmops-examples/
├── README.md
├── helmops.yaml                  # HelmOp CRD configuration
└── charts/
    └── nginx/
        ├── Chart.yaml            # Chart metadata
        ├── values.yaml           # Default values
        └── templates/
            ├── _helpers.tpl      # Template helpers
            ├── deployment.yaml   # Deployment manifest
            ├── service.yaml      # Service manifest
            ├── serviceaccount.yaml
            ├── ingress.yaml      # Ingress configuration
            ├── httproute.yaml    # Gateway API HTTPRoute
            ├── hpa.yaml          # Horizontal Pod Autoscaler
            └── NOTES.txt
```

## HelmOp Configuration

The `helmops.yaml` file defines a HelmOp resource for Fleet deployment:

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: HelmOp
metadata:
  name: nginx
  namespace: fleet-default
spec:
  namespace: default
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
```

### Key Configuration Fields

| Field | Description |
|-------|-------------|
| `apiVersion` | Fleet API version (`fleet.cattle.io/v1alpha1`) |
| `kind` | Resource type (`HelmOp`) |
| `metadata.name` | Name of the HelmOp resource |
| `metadata.namespace` | Namespace where HelmOp is created (typically `fleet-default`) |
| `spec.namespace` | Target namespace for Helm release deployment |
| `spec.helm.releaseName` | Name for the Helm release |
| `spec.helm.chart` | Chart name in the repository |
| `spec.helm.repo` | URL of the Helm repository |
| `spec.targets` | List of target clusters with per-cluster configurations |

### Targets Configuration

Targets specify which clusters to deploy to and allow per-cluster value overrides:

```yaml
targets:
- name: <target-name>           # Unique identifier for this target
  clusterName: "<cluster-name>" # Exact name of the registered cluster
  helm:
    values:                     # Per-cluster Helm value overrides
      replicaCount: 1
```



## Advanced Configuration

### Specifying Chart Version

```yaml
spec:
  helm:
    chart: nginx
    repo: https://rajivreddy.github.io/helmops-examples/
    version: "0.1.0"
```

### Default Values for All Targets

```yaml
spec:
  helm:
    releaseName: nginx
    chart: nginx
    repo: https://rajivreddy.github.io/helmops-examples/
    values:
      image:
        tag: "latest"
      service:
        type: ClusterIP
  targets:
  - name: k3s
    clusterName: "k3s"
    helm:
      values:
        replicaCount: 1  # Overrides default for this cluster
```

### Multiple Cluster Deployments

```yaml
targets:
- name: cluster-1
  clusterName: "k3s"
  helm:
    values:
      replicaCount: 1
- name: cluster-2
  clusterName: "prom-agent"
  helm:
    values:
      replicaCount: 2
- name: cluster-3
  clusterName: "production-cluster"
  helm:
    values:
      replicaCount: 5
      resources:
        limits:
          cpu: 500m
          memory: 256Mi
```

