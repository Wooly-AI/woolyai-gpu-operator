# WoolyAI GPU Operator

This repository is used for storing the WoolyAI GPU Operator HELM Chart.

## Installation

```bash
helm repo add woolyai https://wooly-ai.github.io/woolyai-gpu-operator
helm repo update
helm install woolyai-gpu-operator woolyai/woolyai-gpu-operator
```

## Uninstallation

```bash
helm uninstall woolyai-gpu-operator

