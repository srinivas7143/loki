# Loki Helm Installation with Resource and GOMEMLIMIT Optimization

This guide will help you install or upgrade Grafana Loki using Helm, optimized with resource limits and memory settings. The configuration sets minimal CPU and memory requirements, ideal for resource-constrained environments.

## Prerequisites

- Helm 3 installed
- A Kubernetes cluster with namespace access
- Internet connection to download the Helm chart

## Helm Installation Command

To install Loki using Helm with the specified CPU and memory constraints, run the following command. This setup is optimized for a lightweight environment by adjusting the CPU and memory limits, and setting the Go runtime memory limit (`GOMEMLIMIT`).

```bash
helm upgrade --install --create-namespace -n ${K8S_NAMESPACE} ${HELM_LOKI_RELEASE} grafana/loki \
  -f https://raw.githubusercontent.com/grafana/loki/main/production/helm/loki/single-binary-values.yaml \
  --set-string 'loki.podLabels.sidecar\.istio\.io/inject=true' \
  --set 'singleBinary.resources.requests.cpu=0.25' \
  --set 'singleBinary.resources.requests.memory=250Mi' \
  --set 'singleBinary.resources.limits.cpu=0.25' \
  --set 'singleBinary.resources.limits.memory=250Mi' \
  --set 'loki.auth_enabled=false' \
  --set 'singleBinary.extraEnv[0].name=GOMEMLIMIT' \
  --set 'singleBinary.extraEnv[0].value=225MiB'
```
