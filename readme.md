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
