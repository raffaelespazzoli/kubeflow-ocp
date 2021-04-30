# Helm install

## Install operators

### Cert Manager Operator

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade -i cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.3.1 \
  --create-namespace \
  --set installCRDs=true
```

### Service Mesh Operators

```sh
helm upgrade -i service-mesh-operators helm/service-mesh-operators -n openshift-operators-redhat --create-namespace
```

### Serverless Operator

```sh
helm upgrade -i serverless-operators helm/serverless-operators -n openshift-serverless --create-namespace
```

### Namespace Configuration Operator

```sh
helm repo add namespace-configuration-operator https://redhat-cop.github.io/namespace-configuration-operator
helm repo update
helm install namespace-configuration-operator namespace-configuration-operator/namespace-configuration-operator -n namespace-configuration-operator --create-namespace
```

### Gatekeeper Operator

```sh
helm upgrade -i gatekeeper-operators helm/gatekeeper-operator
```

### Proactive Node Scaling Operator

```sh
helm repo add proactive-node-scaling-operator https://redhat-cop.github.io/proactive-node-scaling-operator
helm repo update
helm install proactive-node-scaling-operator proactive-node-scaling-operator/proactive-node-scaling-operator -n proactive-node-scaling-operator --create-namespace
```

### GPU Operator

```sh
helm upgrade -i gpu-operator helm/gpu-operator -n openshift-operators
```

Create the gpu-operator-resources namespace...

```sh
oc new-project gpu-operator-resources
```

### NFD Operator

```sh
helm upgrade -i nfd-operator helm/nfd-operator -n openshift-operators
```

## Create Autoscaler, ClusterPolicy, and GPU MachineSets

```sh
helm upgrade -i autoscaler helm/autoscaler --set machinesets.infrastructure_id=$(oc get -o jsonpath='{.status.infrastructureName}{"\n"}' infrastructure cluster) -n gpu-operator-resources
```

## Enable kubeflow user's namespaces to schedule on the tainted nodes

```shell
oc apply -f ./openshift/gatekeeper.yaml
oc apply -f ./openshift/ai-ml-toleration.yaml
```
