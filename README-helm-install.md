# Helm install

Steps for installing kubeflow using Helm.

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
helm upgrade -i namespace-configuration-operator namespace-configuration-operator/namespace-configuration-operator -n namespace-configuration-operator --create-namespace
```

### Gatekeeper Operator

```sh
helm upgrade -i gatekeeper-operators helm/gatekeeper-operator
```

### Proactive Node Scaling Operator

```sh
helm repo add proactive-node-scaling-operator https://redhat-cop.github.io/proactive-node-scaling-operator
helm repo update
helm upgrade -i proactive-node-scaling-operator proactive-node-scaling-operator/proactive-node-scaling-operator -n proactive-node-scaling-operator --create-namespace
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
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change
export kubeflow_namespace=kubeflow
export base_domain=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
helm upgrade -i control-plane helm/control-plane -n istio-system --create-namespace --set control_plane.name=${sm_cp_namespace} --set control_plane.name=${sm_cp_name} --set control_plane.ingress.kubeflow.namespace=${kubeflow_namespace} --set base_domain=${base_domain}
```

## Create Knative Serving

```sh
helm upgrade -i serverless helm/serverless -n knative-serving
```

## Integrate ServiceMesh and Serverless

```sh
oc label namespace knative-serving serving.knative.openshift.io/system-namespace=true
oc label namespace knative-serving-ingress serving.knative.openshift.io/system-namespace=true
```

## Enable AWS STS Integration

This is needed only when running in AWS and allows to use workload related credentials (as opposed to individual related), with short lived tokens.

Prepare OIDC endpoint

```shell
oc get -n openshift-kube-apiserver cm -o json bound-sa-token-signing-certs | jq -r '.data["service-account-001.pub"]' > /tmp/sa-signer-pkcs8.pub
./openshift/bin/self-hosted-linux -key "/tmp/sa-signer-pkcs8.pub" | jq '.keys += [.keys[0]] | .keys[1].kid = ""' > "/tmp/keys.json"
export cluster_name=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
export region=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.aws.region}')
export http_location=$(aws s3api create-bucket --bucket oidc-discovery-${cluster_name} --region ${region} --create-bucket-configuration LocationConstraint=${region} | jq -r .Location | sed 's:/*$::') 
export oidc_hostname="${http_location#http://}" 
export oidc_url=https://${oidc_hostname}
envsubst < ./openshift/oidc.json > /tmp/discovery.json
aws s3api put-object --bucket oidc-discovery-${cluster_name} --key keys.json --body /tmp/keys.json
aws s3api put-object --bucket oidc-discovery-${cluster_name} --key '.well-known/openid-configuration' --body /tmp/discovery.json
aws s3api put-object-acl --bucket oidc-discovery-${cluster_name} --key keys.json --acl public-read
aws s3api put-object-acl --bucket oidc-discovery-${cluster_name} --key '.well-known/openid-configuration' --acl public-read
oc patch authentication.config.openshift.io cluster --type "json" -p="[{\"op\": \"replace\", \"path\": \"/spec/serviceAccountIssuer\", \"value\":\"${oidc_url}\"}]"
```

Establish STS trust

```shell
export policy_arn=$(aws iam create-policy --policy-name AllowS3Access --policy-document file://./aws-sts/aws-s3-access-policy.json | jq -r .Policy.Arn)
export thumbprint=$(openssl s_client -showcerts -servername ${oidc_hostname} -connect ${oidc_hostname}:443 </dev/null 2>/dev/null | openssl x509 -outform PEM | openssl x509 -fingerprint -noout)
export thumbprint="${thumbprint##*=}"
export thumbprint="${thumbprint//:}"
export oidc_arn=$(aws iam create-open-id-connect-provider --url ${oidc_url} --client-id-list sts.amazonaws.com --thumbprint-list ${thumbprint} | jq -r .OpenIDConnectProviderArn)
envsubst < ./aws-sts/aws-s3-access-trust-role-policy.json > /tmp/trust-policy.json
export role_arn=$(aws iam create-role --role-name s3-access --assume-role-policy-document file:///tmp/trust-policy.json | jq -r .Role.Arn)
aws iam attach-role-policy --role-name s3-access --policy-arn ${policy_arn}
```

Configure sts injection

```shell
export role_arn=$(aws iam get-role --role-name s3-access | jq -r .Role.Arn)
envsubst < ./openshift/sts-injection.yaml | oc apply -f -
```

## Prepare the kubeflow namespace

```shell
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change

helm upgrade -i kubeflow helm/kubeflow -n kubeflow --create-namespace \
  --set control_plane.namespace=${sm_cp_namespace} \
  --set control_plane.name=${sm_cp_name}

oc label namespace kubeflow istio-injection=enabled control-plane=kubeflow katib-metricscollector-injection=enabled --overwrite

oc adm policy add-scc-to-user anyuid -z kubeflow-pipelines-cache-deployer-sa -n kubeflow
oc adm policy add-scc-to-user anyuid -z xgboost-operator-service-account -n kubeflow
```

## (Optional) Regenerate the kustomize manifests within helm charts

(Optional) - regenerate kustomize output in chart

```sh
kustomize --load-restrictor=LoadRestrictionsNone build ./kubeflow1.3 > helm/kubeflow/templates/kustomize-out.yaml
```

You then need to extract the CRDs from helm/kubeflow/templates/kustomize-out.yaml and put them into helm/kubeflow-crds/templates/crds.yaml so that we can install the CRDs first.

Additionally, theres a ConfigMap that needs be modified to print correctly (not be interpreted by go templating) within helm/kubeflow/templates/kustomize-out.yaml...

See <https://github.com/trevorbox/kubeflow-ocp/blob/helmify/helm/kubeflow/templates/kustomize-out.yaml#L5707>

## Deploy Kubeflow

```shell
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change

helm upgrade -i kubeflow-crds helm/kubeflow-crds -n kubeflow --create-namespace

helm upgrade -i kubeflow helm/kubeflow -n kubeflow --create-namespace \
  --set control_plane.namespace=${sm_cp_namespace} \
  --set control_plane.name=${sm_cp_name}
```

## Configure automatic profile creation, and profile namespace configuration

```shell
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change

helm upgrade -i kubeflow-profile helm/kubeflow-profile -n kubeflow

envsubst < ./openshift/kubeflow-profile-creation.yaml | oc apply -f -
envsubst < ./openshift/kubeflow-profile-namespace-config.yaml | oc apply -f -
```

## Test the kubeflow dashboard

```sh
echo "Navigate to https://$(oc get route secure-kubeflow -n istio-system -o jsonpath={.spec.host})"
```

## Uninstall

```sh
helm delete kubeflow-profile -n kubeflow
helm delete kubeflow -n kubeflow
helm delete kubeflow-crds -n kubeflow
helm delete serverless -n knative-serving
helm delete control-plane -n istio-system
helm delete gatekeeper-configs -n gpu-operator-resources
helm delete gatekeeper-metadata -n gpu-operator-resources
helm delete autoscaler -n gpu-operator-resources
helm delete nfd-operator -n openshift-operators
helm delete gpu-operator -n openshift-operators
helm delete proactive-node-scaling-operator -n proactive-node-scaling-operator
helm delete gatekeeper-operators -n default
helm delete namespace-configuration-operator -n namespace-configuration-operator
helm delete serverless-operators -n openshift-serverless
helm delete service-mesh-operators -n openshift-operators-redhat
helm delete cert-manager -n cert-manager
```
