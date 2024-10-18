# Loki and Grafana Setup on Kyma

This document outlines the steps to set up Loki for log aggregation and Grafana for monitoring and visualization within a Kyma environment.

## Table of Contents

1. [Prerequisites](#prerequisites)

2. [Preparation](#preparation)

3. [Installing Loki](#installing-loki)

4. [Setting Up Log Agent](#setting-up-log-agent)

5. [Installing Grafana](#installing-grafana)

6. [Exposing Grafana](#exposing-grafana)

7. [Adding Grafana to Kyma Dashboard](#adding-grafana-to-kyma-dashboard)

## Prerequisites

- A running Kyma cluster.

- `kubectl` installed and configured to interact with your cluster.

- `helm` installed for package management.

## Preparation

### 1. Set Namespace and Helm Release Name

Set the Kubernetes namespace where Loki and Grafana will be installed.

```bash

export K8S_NAMESPACE="loki" # The namespace for Loki installation

export HELM_LOKI_RELEASE="loki" # The name for the Helm release of Loki

```


**2\. Update Helm Repository**

Add the Grafana Helm chart repository and update it to ensure you have the latest charts.

```bash
helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

```

## Installing Loki

**3\. Install Loki**

Use Helm to install Loki with specific configurations:

```bash
helm upgrade --install --create-namespace -n ${K8S_NAMESPACE} ${HELM_LOKI_RELEASE} grafana/loki

 -f https://raw.githubusercontent.com/grafana/loki/main/production/helm/loki/single-binary-values.yaml

--set-string 'loki.podLabels.sidecar\.istio\.io/inject=true'

--set 'singleBinary.resources.requests.cpu=0.25'

--set 'singleBinary.resources.requests.memory=250Mi'

--set 'singleBinary.resources.limits.cpu=0.25'

--set 'singleBinary.resources.limits.memory=250Mi'

--set 'loki.auth_enabled=false'

--set 'singleBinary.extraEnv[0].name=GOMEMLIMIT'

--set 'singleBinary.extraEnv[0].value=225MiB'
```

**4\. Verify Loki Installation**

Check if the Loki pod is running successfully:
```bash

kubectl -n ${K8S_NAMESPACE} get pod -l app.kubernetes.io/name=loki
```

## Setting Up Log Agent

**5\. Create LogPipeline for Fluent Bit**

Create a LogPipeline resource that forwards logs to Loki:

```bash
cat <<EOF | kubectl apply -f -

apiVersion: telemetry.kyma-project.io/v1alpha1

kind: LogPipeline

metadata:

 name: custom-loki

spec:

 input:

 application:

 namespaces:

 system: true

 output:

 custom: |

 name  loki

 host  ${HELM_LOKI_RELEASE}.${K8S_NAMESPACE}.svc.cluster.local

 port  3100

 auto_kubernetes_labels off

 labels job=fluentbit, container=\$kubernetes['container_name'], namespace=\$kubernetes['namespace_name'], pod=\$kubernetes['pod_name'], node=\$kubernetes['host'], app=\$kubernetes['labels']['app'],app=\$kubernetes['labels']['app.kubernetes.io/name']

EOF
```

## Installing Grafana

**6\. Install Grafana**

Install Grafana using Helm with specified values:

```bash
helm upgrade --install --create-namespace -n ${K8S_NAMESPACE} grafana grafana/grafana -f https://raw.githubusercontent.com/kyma-project/telemetry-manager/main/docs/user/integration/loki/grafana-values.yaml
```

**7\. Enable Loki as Grafana Data Source**

Create a ConfigMap to configure Loki as a data source for Grafana:

```bash
cat <<EOF | kubectl apply -n ${K8S_NAMESPACE} -f -

apiVersion: v1

kind: ConfigMap

metadata:

 name: grafana-loki-datasource

 labels:

 grafana_datasource: "1"

data:

 loki-datasource.yaml: |-

 apiVersion: 1

 datasources:

 - name: Loki

 type: loki

 access: proxy

 url: "http://${HELM_LOKI_RELEASE}:3100"

 version: 1

 isDefault: false

 jsonData: {}

EOF
```

**8\. Retrieve Grafana Admin Password**

To access the Grafana UI, retrieve the admin password:

```bash

kubectl get secret --namespace ${K8S_NAMESPACE} grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

**9\. Port-Forward Grafana Service**

Forward the Grafana service to access the UI locally:

```bash
kubectl -n ${K8S_NAMESPACE} port-forward svc/grafana 3000:80
```
## Exposing Grafana

**10\. Expose Grafana Using Kyma API Gateway**

Create an APIRule to expose Grafana through the Kyma API Gateway:

```bash
cat <<EOF | kubectl apply -f -

apiVersion: gateway.kyma-project.io/v1beta1

kind: APIRule

metadata:

 name: grafana

 namespace: ${K8S_NAMESPACE}

spec:

 corsPolicy:

 allowMethods:

 - GET

 - POST

 gateway: kyma-system/kyma-gateway

 host: my-grafana.c-124cd24.kyma.ondemand.com # Replace with your public URL

 rules:

 - accessStrategies:

 - handler: allow # Allowing all access

 methods:

 - GET

 - POST  # Allowing GET and POST methods for Grafana UI

 mutators:

 - handler: noop # No additional mutation needed

 path: /.*  # Exposing all paths

 service:

 name: grafana  # The service to expose

 port: 80 # Grafana is running on port 80

EOF
```

**11\. Get the Public URL of Grafana**

After applying the APIRule, get the external host of your Grafana service:

```bash

kubectl -n ${K8S_NAMESPACE} get virtualservice -l apirule.gateway.kyma-project.io/v1beta1=grafana.${K8S_NAMESPACE} -ojsonpath='{.items[*].spec.hosts[*]}'

```

Sample output will be like:

my-grafana.c-124cd24.kyma.ondemand.com%

**12\. Save Grafana Public URL as an Environment Variable**

Store the Grafana public URL in an environment variable for later use:
```bash

export GRAFANA_LINK="https://{Virtual-service}" # Replace with your actual host

```

## Adding Grafana to Kyma Dashboard

**13\. Add Grafana Link to Kyma Dashboard**

Add a custom link to the Kyma dashboard for easier access to the Grafana UI:

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

 link: $GRAFANA_LINK # The link to Grafana exposed via the Kyma API Gateway

EOF
```
