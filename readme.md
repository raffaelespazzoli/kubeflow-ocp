# Kubeflow for OpenShift

This guides helps you set up Kubeflow in an existing environment.
We assume that that the following operators are installed:

* [Red Hat ServiceMesh](https://docs.openshift.com/container-platform/4.7/service_mesh/v2x/installing-ossm.html)
* [cert-manager](https://cert-manager.io/docs/installation/openshift/)
* [Red Hat Serverless](https://docs.openshift.com/container-platform/4.7/serverless/admin_guide/installing-openshift-serverless.html)
* [namespace-configuration-operator](https://github.com/redhat-cop/namespace-configuration-operator)
* [gatekeeper-operator](https://github.com/gatekeeper/gatekeeper-operator)
* [proactive-node-autoscaling](https://github.com/redhat-cop/proactive-node-scaling-operator)

We assume that there is a ServiceMesh control plane fully dedicated to the Kubeflow workloads.

This approach sets up Single Sign On (SSO) between Kubeflow and OpenShift.
When an OCP User is created the corresponding Kubeflow profile is also created.

This approach also assumes that the data lake is on AWS S3 and sets up transparent OCP workload to AWS STS authentication.

## Status of the project

| Feature  | Status  |
|:-|:-:|
| Kubeflow - ServiceMesh - Serverless integration  | done  |
| Kubeflow - OCP SSO   | done  |
| Kubeflow multitenancy (this includes automatic profile creation)  | done  |
| Kubeflow notebooks   | done  |
| Elyra notebook  | done  |
| Kubeflow pipelines   | not working, help wanted  |
| kubeflow serving: kfserving | done  |
| kubeflow serving: seldon | done  |
| kubeflow hyperparameters tuning | installed not tested  |
| gpu support: notebooks | done |
| gpu support: training   | not started |
| gpu nodes: autoscaling | done |
| gpu nodes: proactive autoscaling | in progress |
| transparent data lake access: AWS STS Integration | done |

## Enable GPUs and node autoscaling

We assume that nfd operator and gpu operator are correctly installed.
We assume that gpu-enabled nodes are provisioned and tainted with:

```yaml
  taints:
    - key: workload
      value: ai-ml
      effect: NoSchedule
```

and labeled `workload=ai-ml`

### Create gpu nodes

You'll have to customize this with the right instance type

```shell
  export cluster_name=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
  export region=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.aws.region}')
  export ami=$(oc get machineset -n openshift-machine-api -o jsonpath='{.items[0].spec.template.spec.providerSpec.value.ami.id}')
  export machine_type=ai-ml
  export instance_type=g3s.xlarge
  for z in a b c; do
    export zone=${region}${z}
    oc scale machineset -n openshift-machine-api $(envsubst < ./openshift/gpu-machineset.yaml | yq -r .metadata.name) --replicas 0 
    envsubst < ./openshift/gpu-machineset.yaml | oc apply -f -
  done
```

### Enable kubeflow user's namespaces to schedule on the tainted nodes

```shell
oc apply -f ./openshift/gatekeeper.yaml
oc apply -f ./openshift/ai-ml-toleration.yaml
```

### Enable Autoscaling on gpu nodes

```shell
oc apply -f ./openshift/autoscaler.yaml
for machineset in $(oc get machineset -n openshift-machine-api -o json | jq -r '.items[] | select (.spec.replicas!=0 and .spec.template.spec.metadata.labels.workload=="ai-ml") | .metadata.name'); do
  machineset=$machineset envsubst < ./openshift/machineset-autoscaler.yaml | oc apply -f - -n openshift-machine-api
done
```

### Enable proactive node autoscaling

```shell
oc apply -f ./openshift/ai-ml-watermark.yaml
```

## Patch the service mesh for Kubeflow SSO

```shell
export allowed_cidrs_csv=$(curl --no-progress-meter https://ip-ranges.amazonaws.com/ip-ranges.json | jq -r '.prefixes[] | select(.region=="us-east-2" or .region=="us-east-1" or .region=="us-west-1" or .region=="us-west-2") | select(.service=="S3" or .service=="AMAZON") | .ip_prefix' | awk -vORS=, '{print $1}' | sed 's/,$/\n/')
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change
export base_domain=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
envsubst < ./openshift/sm_cp_patch.yaml | oc apply -f - -n ${sm_cp_namespace}
```

## Integrate ServiceMesh and Serverless

```shell
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
oc new-project kubeflow
oc label namespace kubeflow  control-plane=kubeflow katib-metricscollector-injection=enabled istio-injection=enabled
envsubst < ./openshift/kubeflow-sm-member.yaml | oc apply -f - -n kubeflow
oc apply -f ./openshift/allow-apiserver-webhooks.yaml -n kubeflow
oc adm policy add-scc-to-user anyuid -z application-controller-service-account -n kubeflow
oc adm policy add-scc-to-user anyuid -z default -n kubeflow
oc adm policy add-scc-to-user anyuid -z seldon-manager -n kubeflow
oc adm policy add-scc-to-user anyuid -z kubeflow-pipelines-cache-deployer-sa -n kubeflow
oc adm policy add-scc-to-user anyuid -z admission-webhook-service-account -n kubeflow
```

deploy kubeflow

```shell
export manifests_dir=/home/rspazzol/git/kubeflow-manifests #change to something else if need be
git -C $(dirname "$manifests_dir") clone https://github.com/raffaelespazzoli/manifests -b ocp
export KF_DIR=$(pwd)/kfctl-run

## loop from here

rm -rf ${KF_DIR}
mkdir -p ${KF_DIR}
envsubst < ./kfctl_kube_multi-user.v1.2.0.yaml  > ./kfctl-run/kfctl_kube_multi-user.v1.2.0.yaml
cd ${KF_DIR}
kfctl build -V --file=./kfctl_kube_multi-user.v1.2.0.yaml
kfctl apply -V --file=./kfctl_kube_multi-user.v1.2.0.yaml
cd ..

## to here
```

## Configure automatic profile creation, and profile namespace configuration:

```shell
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change
oc apply -f ./openshift/kubeflow-profile-creation.yaml
envsubst < ./openshift/kubeflow-profile-namespace-config.yaml | oc apply -f -
```

## Prepare notebook image compatible with OCP security

Kubeflow notebook images have some special requirements, this is just an example of how to create one.

```shell
oc apply -f ./openshift/build-notebook-image.yaml -n openshift
oc start-build tf-notebook-image -n openshift

this will create this image:
image-registry.openshift-image-registry.svc:5000/openshift/tf-notebook-image
```

TODO add instructions on how to prepare the elyra image.

## Kubeflow serving: kfserving

Position yourself in a user namespace

```shell
export namespace=raffa
oc apply -f ./serving/kfserving/tensorflow/tf-inference.yaml -n ${namespace}
```

verifying the result

```shell
MODEL_NAME=flowers-sample
INPUT_PATH=@./serving/kfserving/tensorflow/input.json
SERVICE_HOSTNAME=$(oc get inferenceservice ${MODEL_NAME} -n ${namespace} -o jsonpath='{.status.default.predictor.host}')
curl -k https://${SERVICE_HOSTNAME}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

## Kubeflow serving: seldon

Position yourself in a user namespace

Mock test

```shell
export namespace=raffa
export ingress_hostame=istio-system-istio-autogenerated-k8s-ingress-xvx5k-istio-system.apps.tmp-raffa.demo.red-chesterfield.com
oc apply -f ./serving/seldon/mock/seldon.yaml -n ${namespace}
curl -s -d '{"data": {"ndarray":[[1.0, 2.0, 5.0]]}}'    -X POST http://istio-system-istio-autogenerated-k8s-ingress-xvx5k-istio-system.apps.tmp-raffa.demo.red-chesterfield.com/seldon/${namespace}/mock-model/api/v1.0/predictions    -H "Content-Type: application/json"
```

```shell
oc apply -f ./serving/seldon/tensorflow/seldon.yaml -n ${namespace}
```

Testing:

```shell
MODEL_NAME=flowers-sample
INPUT_PATH=@./serving/seldon/tensorflow/input.json
SERVICE_HOSTNAME=$(oc get route istio-ingressgateway -n istio-system -o jsonpath='{.spec.host}')
curl -k http://${SERVICE_HOSTNAME}/seldon/${namespace}/${MODEL_NAME}/api/v1.0/predictions -d $INPUT_PATH  -H "Content-Type: application/json"
```



## remove kubeflow

```shell
kfctl delete -V --file=./kfctl_kube_multi-user.v1.2.0.yaml
```
