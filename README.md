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
kubectl create namespace woolyai-gpu-operator

# NOTE: Do not deploy anything else in this namespace.

# 3. Create the license secret
kubectl -n woolyai-gpu-operator create secret generic woolyai-license \
  --from-file=license.json=license.json

# 4. Label your GPU nodes
kubectl label node 149-118-79-27 gpu-runtime=woolyai --overwrite

# 5. Taint nodes to prevent NVIDIA/WoolyAI conflicts
kubectl taint node 149-118-79-27 woolyai.com/runtime=true:NoSchedule

# 6. Install the operator
# Note: You can override the server image to use a different version by setting the controller.server.image.override parameter.
# --set controller.server.image.override=woolyai/server:cuda13.1.1-latest \
helm install woolyai-gpu-operator woolyai/woolyai-gpu-operator \
  --set licenseSecretName=woolyai-license \
  --namespace woolyai-gpu-operator \
  --wait --timeout 300s

# View the values that were used to install the operator
helm get values woolyai-gpu-operator -n woolyai-gpu-operator
```

> **Note**: Find available server image tags at https://hub.docker.com/r/woolyai/server/tags

### Preventing NVIDIA/WoolyAI Conflicts

WoolyAI expects to be the only GPU scheduler on nodes labeled `gpu-runtime=woolyai`. If you're running a mixed cluster with both WoolyAI and standard NVIDIA GPU workloads, taint your WoolyAI nodes to prevent conflicts:

```bash
kubectl taint node <node-name> woolyai.com/runtime=true:NoSchedule
```

The WoolyAI admission webhook automatically injects a toleration for `woolyai.com/runtime` into WoolyAI-managed pods, so they continue to schedule normally. Standard NVIDIA GPU workloads (without the toleration) will be kept off these nodes.

To remove the taint later:

```bash
kubectl taint node <node-name> woolyai.com/runtime=true:NoSchedule-
```

### Verify Installation

```bash
kubectl get pods -n woolyai-gpu-operator
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

### Upgrading

To upgrade the WoolyAI GPU Operator to a newer version:

```bash
# 1. Update the Helm repository
helm repo update woolyai

# 2. Upgrade the operator (use the same overrides you used for the initial install)
helm upgrade woolyai-gpu-operator woolyai/woolyai-gpu-operator \
  --set licenseSecretName=woolyai-license \
  --namespace woolyai-gpu-operator \
  --wait --timeout 300s

# The new chart version references version-pinned image tags, so Kubernetes
# automatically pulls the correct images. No pullPolicy changes needed.
```

> **Note**: Each chart version pins image tags to its `appVersion` (e.g., `controller-0.1.4`). This means `helm upgrade` to a new chart version automatically triggers image pulls without needing to set `pullPolicy=Always`. Existing pods are safe from unexpected version drift on restarts.

### Rolling Back or Pinning a Specific Image Version

Each chart version automatically uses image tags matching its `appVersion`. If you need to roll back to a previous chart release:

```bash
# Roll back to the previous Helm release
helm rollback woolyai-gpu-operator -n woolyai-gpu-operator
```

To install or pin a specific chart version:

```bash
helm upgrade woolyai-gpu-operator woolyai/woolyai-gpu-operator \
  --version 0.1.4 \
  --set licenseSecretName=woolyai-license \
  --namespace woolyai-gpu-operator \
  --wait --timeout 300s
```

If you need to override image versions independently of the chart version (e.g., to test a specific build), use `imageTagSuffix`:

```bash
# Use a specific image version across all components
helm upgrade woolyai-gpu-operator woolyai/woolyai-gpu-operator \
  --set licenseSecretName=woolyai-license \
  --set imageTagSuffix=0.1.1 \
  --namespace woolyai-gpu-operator \
  --wait --timeout 300s
```

This overrides all image tags (e.g., `controller-0.1.1`, `admission-0.1.1`, etc.) without changing the chart version itself. You can also override individual component images using the `image.override` field:

```bash
# Override a single component image
helm upgrade woolyai-gpu-operator woolyai/woolyai-gpu-operator \
  --set licenseSecretName=woolyai-license \
  --set admission.libInjector.image.override=woolyai/gpu-operator:lib-injector-0.1.1 \
  --namespace woolyai-gpu-operator \
  --wait --timeout 300s
```

---

## Usage Guide

### Deploying GPU Workloads

Pods request shared GPUs using the `gpu-runtime: woolyai` label and `woolyai.com/gpu` resource and annotations.

#### Minimal Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
  labels:
    gpu-runtime: woolyai  # Required
  annotations:
    woolyai.com/pod-vram: "11Gi"  # Required: VRAM needed per GPU
spec:
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
  labels:
    gpu-runtime: woolyai  # Required
  annotations:
    woolyai.com/pod-vram: "16000Mi"      # Required VRAM per GPU (MiB)
    woolyai.com/priority: "1"             # Priority 0-4 (0 = highest)
    woolyai.com/exclusive-gpu-use: "false" # Set true for dedicated GPU
    woolyai.com/swap-from-vram: "true"    # Allow VRAM spill to host memory
spec:
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

### How It Differs from other GPU Scheduling

Other GPU schedulers allocate **whole GPUs** as discrete resources (e.g., `nvidia.com/gpu: 1`). The WoolyAI scheduler instead:

- **Schedules by VRAM capacity**, not GPU count — pods request memory like `woolyai.com/pod-vram: 16Gi`
- **Assigns specific GPU IDs** on the node, not just "some GPU"
- **Uses real-time inventory** from `NodeGPUStatus` CRs, not static `Allocatable` counts

### Scheduling Pipeline

The scheduler runs through these phases:

| Phase | Description |
|-------|-------------|
| **QueueSort** | Orders pending pods by `woolyai.com/priority` (0 = highest) |
| **PreFilter** | Parses pod annotations (`pod-vram`, `priority`, `gpu-class`) |
| **Filter** | Eliminates nodes that can't satisfy the request (wrong GPU class, insufficient VRAM, exclusive constraints) |
| **Score** | Ranks surviving nodes — prefers idle GPUs, avoids contention, considers GPU starvation |
| **Reserve** | Picks specific GPU IDs and patches the pod with assignments (only if `woolyai.com/gpu-ids` is set) |
| **Permit** | Confirms reservation with node server |

### GPU Assignment Output

After scheduling, the pod is patched with:

```yaml
woolyai.com/gpu-ids: "0,2"              # Physical GPU indices assigned
woolyai.com/reserved-vram-mib: "16384"  # VRAM slice promised
woolyai.com/reservation-uid: "abc123"   # Unique reservation ID
```

The admission webhook converts these into environment variables (`WOOLYAI_GPU_IDS`, `WOOLYAI_RESERVED_VRAM_MIB`, etc.) via Downward API, so the WoolyAI client library knows which GPUs to use without calling Kubernetes APIs.

---

## Troubleshooting

### Check Deployment Status

```bash
# View all pods
kubectl get pods -n woolyai-gpu-operator

# View deployments
kubectl get deployments -n woolyai-gpu-operator

# View daemonsets (server pods)
kubectl get daemonsets -n woolyai-gpu-operator
```

### Diagnose Pod Issues

```bash
# Describe a specific pod
kubectl describe pod <pod-name> -n woolyai-gpu-operator

# Check logs for operator components
kubectl logs -n woolyai-gpu-operator -l app.kubernetes.io/name=woolyai-gpu-operator-controller --tail=100
kubectl logs -n woolyai-gpu-operator -l app.kubernetes.io/name=woolyai-gpu-operator-admission --tail=100
kubectl logs -n woolyai-gpu-operator -l app.kubernetes.io/name=woolyai-gpu-operator-scheduler --tail=100

# Check server pod logs
kubectl logs -n woolyai-gpu-operator <woolyai-server-pod-name> --tail=100

# View recent events
kubectl get events -n woolyai-gpu-operator --sort-by='.lastTimestamp'
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
kubectl delete daemonset -l app.kubernetes.io/component=server -n woolyai-gpu-operator

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
kubectl get pods -n woolyai-gpu-operator

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
# 1. Remove node labels and taints so server is removed from the nodes
for node in $(kubectl get nodes -l gpu-runtime=woolyai -o jsonpath='{.items[*].metadata.name}'); do
  echo "Removing labels and taints from $node"
  kubectl label node "$node" \
    gpu-runtime- \
    woolyai.com/gpu.present- \
    woolyai.com/gpu.count- \
    woolyai.com/gpu.vram-
  kubectl taint node "$node" woolyai.com/runtime=true:NoSchedule- 2>/dev/null || true
done

# 2. Uninstall the Helm release
helm uninstall woolyai-gpu-operator -n woolyai-gpu-operator

# 3. Clean up cluster-scoped and cross-namespace resources (not always removed by helm uninstall)
kubectl delete clusterrole -l app.kubernetes.io/instance=woolyai-gpu-operator 2>/dev/null || true
kubectl delete clusterrolebinding -l app.kubernetes.io/instance=woolyai-gpu-operator 2>/dev/null || true
kubectl delete rolebinding -n kube-system -l app.kubernetes.io/instance=woolyai-gpu-operator 2>/dev/null || true
kubectl delete mutatingwebhookconfiguration woolyai-admission-mutating 2>/dev/null || true
kubectl delete validatingwebhookconfiguration woolyai-admission-validating 2>/dev/null || true

# 4. Delete any remaining CRD instances
kubectl delete woolynodepolicies --all 2>/dev/null || true
kubectl delete clustergpupolicies --all 2>/dev/null || true
kubectl delete gpuclasses --all 2>/dev/null || true
kubectl delete nodegpustatuses --all 2>/dev/null || true

# 5. Delete the CRDs themselves
kubectl delete crd woolynodepolicies.woolyai.dev 2>/dev/null || true
kubectl delete crd clustergpupolicies.woolyai.dev 2>/dev/null || true
kubectl delete crd gpuclasses.woolyai.dev 2>/dev/null || true
kubectl delete crd nodegpustatuses.woolyai.dev 2>/dev/null || true

# 6. (Optional) Remove cached WoolyAI images from nodes
sudo crictl images | grep woolyai | awk '{print $3}' | xargs -r sudo crictl rmi

# 7. Delete the namespace
kubectl delete namespace woolyai-gpu-operator
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
kubectl logs -n woolyai-gpu-operator -l app.kubernetes.io/name=woolyai-server -c woolyai-server --follow
```
