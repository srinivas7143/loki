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
```bash
kubectl apply -f - <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: grafana
  namespace: kube-public
  labels:
    busola.io/extension: statics
    busola.io/extension-version: '0.5'
data:
  general: |-
    externalNodes:
    - category: Observability
      icon: display
      children:
      - label: My Grafana
        link: https://my-grafana.c-124cd24.kyma.ondemand.com
EOF
```
```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.kyma-project.io/v1beta1
kind: APIRule
metadata:
  name: grafana
  namespace: loki
spec:
  corsPolicy:
    allowMethods:
      - GET
      - POST
  gateway: kyma-system/kyma-gateway
  host: my-grafana
  rules:
    - accessStrategies:
        - handler: allow
      methods:
        - GET
        - POST
      mutators:
        - handler: noop
      path: /.*
  service:
    name: grafana
    port: 80
EOF
```
