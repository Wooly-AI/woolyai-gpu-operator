# WoolyAI GPU Operator

A Kubernetes-native GPU operator that enables intelligent GPU sharing and multiplexing across workloads in a cluster.

## Overview

[WoolyAI](https://woolyai.com/) allows multiple Kubernetes pods to **share physical GPUs** through a per-node WoolyAI Server that multiplexes CUDA command streams from multiple clients. This enables higher GPU utilization compared to traditional exclusive GPU allocation.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Control Plane                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Scheduler   │  │  Controller  │  │ Admission Webhook│  │
│  │   Plugin     │  │  (Operator)  │  │  (Pod Mutation)  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Node Components                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ WoolyAI      │  │ Node Agent   │  │ Device Plugin    │  │
│  │ Server       │  │ (DaemonSet)  │  │ (GPU Advertise)  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Start

### Prerequisites

- Kubernetes v1.24+
- NVIDIA GPU nodes with drivers installed
- A valid WoolyAI license file (`license.json`) --Contact support@woolyai.com to get one.

### Installation

```bash
# 1. Add the Helm repository
helm repo add woolyai https://wooly-ai.github.io/woolyai-gpu-operator
helm repo update

# 2. Create the namespace
kubectl create namespace woolyai

# 3. Create the license secret
kubectl -n woolyai create secret generic woolyai-license \
  --from-file=license.json=license.json

# 4. Label your GPU nodes
kubectl label node <node-name> gpu-runtime=woolyai --overwrite
# Example:
# kubectl label node l4-1 gpu-runtime=woolyai --overwrite
# kubectl label node l4-2 gpu-runtime=woolyai --overwrite

# 5. (Optional) Taint nodes to prevent NVIDIA/WoolyAI conflicts
kubectl taint node <node-name> woolyai.com/runtime=true:NoSchedule
# Example:
# kubectl taint node l4-1 woolyai.com/runtime=true:NoSchedule
# kubectl taint node l4-2 woolyai.com/runtime=true:NoSchedule

# 6. Install the operator
helm install woolyai-gpu-operator woolyai/woolyai-gpu-operator \
  --set licenseSecretName=woolyai-license \
  --namespace woolyai \
  --wait --timeout 300s

# Add this if you want to use something other than the latest server image
# --set controller.server.image.override=woolyai/server:cuda13.1.1-latest \
```

> **Note**: Find available server image tags at https://hub.docker.com/r/woolyai/server/tags

### Image Pull Policy

By default, Kubernetes uses `IfNotPresent` for image pull policy. If you're using a `latest` tag or want to ensure you always pull the newest image (e.g., during development or after a server update), set the pull policy to `Always`:

```bash
helm install woolyai-gpu-operator woolyai/woolyai-gpu-operator \
  --set controller.server.image.override=woolyai/server:cuda13.1.1-latest \
  --set controller.server.image.pullPolicy=Always \
  --set licenseSecretName=woolyai-license \
  --namespace woolyai \
  --wait --timeout 300s
```

To update an existing installation with the new pull policy:

```bash
helm upgrade woolyai-gpu-operator woolyai/woolyai-gpu-operator \
  --set controller.server.image.override=woolyai/server:cuda13.1.1-latest \
  --set controller.server.image.pullPolicy=Always \
  --set licenseSecretName=woolyai-license \
  --namespace woolyai \
  --wait --timeout 300s
```

> **Tip**: After changing the pull policy, you may need to restart the server pods to pull the latest image:
> ```bash
> kubectl delete daemonset woolyai-server-auto-l4-1 woolyai-server-auto-l4-2 -n woolyai
> kubectl delete woolynodepolicies --all
> ```

### Verify Installation

```bash
kubectl get pods -n woolyai
```

Expected output:
```
NAME                                              READY   STATUS    RESTARTS   AGE
woolyai-gpu-operator-admission-6db5ffb79c-qtkvt   1/1     Running   0          60s
woolyai-gpu-operator-controller-86d4b5cc5-cmqwt   1/1     Running   0          60s
woolyai-gpu-operator-scheduler-6c78c56cf5-p62xs   1/1     Running   0          60s
woolyai-server-auto-l4-1-7w7s8                    4/4     Running   0          55s
woolyai-server-auto-l4-2-wqpws                    4/4     Running   0          55s
```

---

## Usage Guide

### Deploying GPU Workloads

Pods request shared GPUs using the `woolyai.com/gpu` resource and annotations.

#### Minimal Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
  annotations:
    woolyai.com/pod-vram: "11Gi"  # Required: VRAM needed per GPU
spec:
  schedulerName: woolyai-scheduler
  containers:
  - name: worker
    image: nvidia/cuda:13.1.1-cudnn-runtime-ubuntu22.04
    resources:
      limits:
        woolyai.com/gpu: "1"
```

#### Full Example with All Options

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload-advanced
  annotations:
    woolyai.com/pod-vram: "16000Mi"      # Required VRAM per GPU (MiB)
    woolyai.com/priority: "1"             # Priority 0-4 (0 = highest)
    woolyai.com/exclusive-gpu-use: "false" # Set true for dedicated GPU
    woolyai.com/swap-from-vram: "true"    # Allow VRAM spill to host memory
    cuda.scheduler.io/gpu-class: "a100-shared"  # Target specific GPU class
spec:
  schedulerName: woolyai-scheduler
  containers:
  - name: worker
    image: nvidia/cuda:13.1.1-cudnn-runtime-ubuntu22.04
    resources:
      limits:
        woolyai.com/gpu: "2"  # Request 2 GPUs
```

### Pod Annotations Reference

#### Required Annotations

| Annotation | Description | Example |
|------------|-------------|---------|
| `woolyai.com/pod-vram` | Required VRAM per GPU (MiB) | `16000Mi` |

#### Optional Annotations

| Annotation | Description | Default |
|------------|-------------|---------|
| `woolyai.com/priority` | Priority level (0-4, where 0 is highest) | `0` |
| `woolyai.com/exclusive-gpu-use` | Dedicated GPU mode (no sharing) | `false` |
| `woolyai.com/swap-from-vram` | Allow VRAM spill to host memory | `true` |
| `cuda.scheduler.io/gpu-class` | Target a specific GPUClass | - |

#### Scheduler-Populated Annotations

These are set automatically by the scheduler:

| Annotation | Description |
|------------|-------------|
| `woolyai.com/gpu-ids` | Comma-separated GPU indices assigned |
| `woolyai.com/reserved-vram-mib` | VRAM slice promised after overcommit |
| `woolyai.com/reservation-uid` | Unique reservation identifier |

---

## Configuration

### Custom Resource Definitions (CRDs)

| CRD | Purpose |
|-----|---------|
| **GPUClass** | Defines GPU profiles (product, MIG layout, sharing mode, min VRAM) |
| **WoolyNodePolicy** | Per-node/nodepool concurrency limits and VRAM overcommit settings |
| **ClusterGPUPolicy** | Cluster-wide defaults for VRAM overcommit |
| **NodeGPUStatus** | Real-time telemetry per node (available VRAM, starvation %, running priorities) |

### ClusterGPUPolicy

Set cluster-wide VRAM overcommit defaults:

```yaml
apiVersion: woolyai.dev/v1alpha1
kind: ClusterGPUPolicy
metadata:
  name: default
spec:
  defaultVramOvercommitPercent: 20
```

### WoolyNodePolicy

Configure per-node concurrency limits:

```yaml
apiVersion: woolyai.dev/v1alpha1
kind: WoolyNodePolicy
metadata:
  name: high-memory-nodes
spec:
  targetSelector:
    node-type: gpu-high-mem
  maxClientsPerGPU: 4
  maxClientsPerNode: 16
  vramOvercommitPercent: 30
```

### GPUClass

Define GPU profiles for workload targeting:

```yaml
apiVersion: woolyai.dev/v1alpha1
kind: GPUClass
metadata:
  name: a100-shared
spec:
  description: "Shared A100 GPUs"
  matchLabels:
    woolyai.com/gpu.product: A100-SXM4-40GB
  requirements:
    minVramMiB: 40000
  sharing:
    mode: shared
    replicas: 4
```

---

## Scheduling

### How It Works

The WoolyAI scheduler plugin uses a multi-phase approach:

1. **QueueSort** - Orders pods by priority (0-4, where 0 is highest)
2. **PreFilter** - Parses annotations (`woolyai.com/pod-vram`, `woolyai.com/priority`, etc.)
3. **Filter** - Validates GPU capacity, VRAM availability with overcommit, exclusivity constraints
4. **Score** - Ranks nodes by idle GPUs, starvation percentage, VRAM headroom
5. **Reserve** - Assigns concrete GPU IDs and persists them to pod annotations
6. **Permit** - Confirms reservation with node server

---

## Troubleshooting

### Check Deployment Status

```bash
# View all pods
kubectl get pods -n woolyai

# View deployments
kubectl get deployments -n woolyai

# View daemonsets (server pods)
kubectl get daemonsets -n woolyai
```

### Diagnose Pod Issues

```bash
# Describe a specific pod
kubectl describe pod <pod-name> -n woolyai

# Check logs for operator components
kubectl logs -n woolyai -l app.kubernetes.io/name=woolyai-gpu-operator-controller --tail=100
kubectl logs -n woolyai -l app.kubernetes.io/name=woolyai-gpu-operator-admission --tail=100
kubectl logs -n woolyai -l app.kubernetes.io/name=woolyai-gpu-operator-scheduler --tail=100

# Check server pod logs
kubectl logs -n woolyai <woolyai-server-pod-name> --tail=100

# View recent events
kubectl get events -n woolyai --sort-by='.lastTimestamp'
```

### Check GPU Status

```bash
# View NodeGPUStatus resources
kubectl get nodegpustatuses -A

# View detailed status for a node
kubectl describe nodegpustatus <node-name>

# Check node labels
kubectl get nodes --show-labels | grep woolyai
```

### Server Pods Disappear

If server pods disappear or get stuck, recreate them:

```bash
# Delete the old DaemonSets
kubectl delete daemonset -l app.kubernetes.io/component=server -n woolyai

# Delete the WoolyNodePolicies to trigger recreation
kubectl delete woolynodepolicies --all

# Re-label the nodes to trigger fresh creation
kubectl label node <node-name> gpu-runtime=woolyai --overwrite
# Example:
# kubectl label node l4-1 gpu-runtime=woolyai --overwrite
# kubectl label node l4-2 gpu-runtime=woolyai --overwrite
```

### Helm Install Timeout

If the Helm install times out:

```bash
# Check what's pending
kubectl get pods -n woolyai

# Describe problematic pods
kubectl describe pod -n woolyai -l app.kubernetes.io/instance=woolyai-gpu-operator

# Common causes:
# - Image pull issues (check image name/registry access)
# - Missing license secret
# - Resource constraints on nodes
# - Nodes not labeled with gpu-runtime=woolyai
```

---

## Uninstallation

### Complete Removal

```bash
# Remove node labels so server is removed from the nodes
for node in $(kubectl get nodes -l gpu-runtime=woolyai -o jsonpath='{.items[*].metadata.name}'); do
  echo "Removing labels from $node"
  kubectl label node "$node" \
    gpu-runtime- \
    woolyai.com/gpu.present- \
    woolyai.com/gpu.count- \
    woolyai.com/gpu.vram-
done

# 1. Uninstall the Helm release
helm uninstall woolyai-gpu-operator -n woolyai

# Remove all WoolyAI images
sudo crictl images | grep woolyai | awk '{print $3}' | xargs -r sudo crictl rmi

# 2. Delete the webhooks (if they persist)
kubectl delete mutatingwebhookconfiguration woolyai-admission-mutating 2>/dev/null || true
kubectl delete validatingwebhookconfiguration woolyai-admission-validating 2>/dev/null || true

# 3. Delete any remaining CRD instances
kubectl delete woolynodepolicies --all
kubectl delete clustergpupolicies --all 2>/dev/null || true
kubectl delete gpuclasses --all 2>/dev/null || true
kubectl delete nodegpustatuses --all 2>/dev/null || true

# 4. Delete the CRDs themselves
kubectl delete crd woolynodepolicies.woolyai.dev
kubectl delete crd clustergpupolicies.woolyai.dev
kubectl delete crd gpuclasses.woolyai.dev
kubectl delete crd nodegpustatuses.woolyai.dev

# 6. Delete the namespace
kubectl delete namespace woolyai
```

---

## Components Reference

| Component | Purpose |
|-----------|---------|
| **Scheduler Plugin** | Custom kube-scheduler plugin for GPU-aware pod placement using VRAM, priority, and exclusivity constraints |
| **Controller** | Manages WoolyAI Server DaemonSets and reconciles policies |
| **Admission Webhook** | Injects env vars (`WOOLYAI_GPU_IDS`, etc.), library shims, and validates pod GPU requests |
| **Device Plugin** | Advertises GPU resources to Kubernetes |
| **Node Agent** | Reports `NodeGPUStatus` CRDs with per-GPU telemetry |
| **WoolyAI Server** | Per-node server that multiplexes CUDA command streams from multiple clients |

---

## Support

For issues and feature requests, please contact WoolyAI support at support@woolyai.com or join our Slack at https://slack.woolyai.com/.

## Troubleshooting

### Check Server Logs:

```bash
kubectl logs -n woolyai -l app.kubernetes.io/name=woolyai-server -c woolyai-server --follow
```