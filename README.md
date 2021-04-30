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

### Get entitlement from Red Hat

Get entitlement...

1. Navigate to <https://access.redhat.com/management/systems>
2. Create a new system, call it whatever you like
3. Select the newly created system
4. Select Subscriptions tab
5. Select Attach Subscription
6. Search and add the "Red Hat Developer Subscription for Individuals" Subscription
7. In the same window select the Download Certificates button
8. Unzip the consumer_export within the zipped file
9. Grab the location of consumer_export/export/entitlement_certificates/*.pem file and copy it to the current directory, naming it `nvidia.pem` instead

Optionally, you may follow the Instructions from <https://docs.nvidia.com/datacenter/kubernetes/openshift-on-gpu-install-guide/index.html#openshift-gpu-install-gpu-operator-via-helmv3> to test the entitlement.

## Create Autoscaler, ClusterPolicy, and GPU MachineSets

```sh
helm upgrade -i autoscaler helm/autoscaler --set machinesets.infrastructure_id=$(oc get -o jsonpath='{.status.infrastructureName}{"\n"}' infrastructure cluster) --set cluster_policy.machineconfigs.base64_pem=$(base64 -w0 nvidia.pem) -n gpu-operator-resources 
```

See <https://github.com/NVIDIA/gpu-operator/issues/140> which explains why we need to set the initial replicas to one for the p2 instance type to avoid constant down and up scaling.

See <https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#my-cluster-is-below-minimum--above-maximum-number-of-nodes-but-ca-did-not-fix-that-why> which explains why the MachineAutoscaler won't increase the minimum to one for the p2 instance type initially... there are no unschedulable pods at this point.

## Enable kubeflow user's namespaces to schedule on the tainted nodes

```shell
helm upgrade -i gatekeeper-configs helm/gatekeeper-configs -n gpu-operator-resources
helm upgrade -i gatekeeper-metadata helm/gatekeeper-metadata -n gpu-operator-resources
```

## Patch the service mesh for Kubeflow SSO

```shell
export allowed_cidrs_csv=$(curl -s https://ip-ranges.amazonaws.com/ip-ranges.json | jq -r '.prefixes[] | select(.region=="us-east-2" or .region=="us-east-1" or .region=="us-west-1" or .region=="us-west-2") | select(.service=="S3" or .service=="AMAZON") | .ip_prefix' | awk -vORS=, '{print $1}' | sed 's/,$/\n/')
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change
export kubeflow_namespace=kubeflow
export base_domain=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
helm upgrade -i control-plane helm/control-plane -n istio-system --create-namespace --set control_plane.allowed_cidrs_csv=${allowed_cidrs_csv} --set control_plane.name=${sm_cp_namespace} --set control_plane.name=${sm_cp_name} --set control_plane.ingress.kubeflow.namespace=${kubeflow_namespace} --set base_domain=${base_domain}
```
