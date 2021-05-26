# Kubeflow for OpenShift

> For installation using Helm, follow [README-helm-install.md](./README-helm-install.md)

This guides helps you set up Kubeflow 1.3 in an existing environment.
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
| Kubeflow pipelines   | needs testing  |
| kubeflow serving: kfserving | done  |
| kubeflow hyperparameters tuning | needs testing  |
| tensorboard | needs testing |
| kubeflow experiments | needs testing |
| kubeflow training kfserving | done |
| kubeflow distributed training pytorch | done |
| gpu support: notebooks | done |
| gpu support: training   | done |
| gpu nodes: autoscaling | done - need to closely observe the autoscaler behavior |
| gpu nodes: proactive autoscaling | done |
| transparent data lake access: AWS STS Integration | done |
| kubeflow pipelines | done |

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
oc apply -f ./openshift/priority-classes.yaml
oc apply -f ./openshift/ai-ml-watermark.yaml
```

## Patch the service mesh for Kubeflow SSO

```shell
export allowed_cidrs_csv=$(curl -s https://ip-ranges.amazonaws.com/ip-ranges.json | jq -r '.prefixes[] | select(.region=="us-east-2" or .region=="us-east-1" or .region=="us-west-1" or .region=="us-west-2") | select(.service=="S3" or .service=="AMAZON") | .ip_prefix' | awk -vORS=, '{print $1}' | sed 's/,$/\n/')
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change
export kubeflow_namespace=kubeflow
export base_domain=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
envsubst < ./openshift/sm_cp_patch.yaml | oc apply -f -
```

## Enable Knative Serving

```sh
oc apply -f openshift/knativeserving-knative-serving.yaml -n knative-serving
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
oc label namespace kubeflow istio-injection=enabled control-plane=kubeflow katib-metricscollector-injection=enabled --overwrite
envsubst < ./openshift/kubeflow-sm-member.yaml | oc apply -f - -n kubeflow
oc apply -f ./openshift/allow-apiserver-webhooks.yaml -n kubeflow
oc adm policy add-scc-to-user anyuid -z kubeflow-pipelines-cache-deployer-sa -n kubeflow
oc adm policy add-scc-to-user anyuid -z xgboost-operator-service-account -n kubeflow
```

deploy kubeflow

> Note: You may need to run this command twice since the CRDs need to exist first.

```shell
./kustomize --load-restrictor=LoadRestrictionsNone build ./kubeflow1.3 | oc apply -f -
```

## Configure automatic profile creation, and profile namespace configuration:

```shell
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change
envsubst < ./openshift/kubeflow-profile-creation.yaml | oc apply -f -
envsubst < ./openshift/kubeflow-profile-namespace-config.yaml | oc apply -f -
```

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
SERVICE_HOSTNAME=$(oc get ksvc flowers-sample-predictor-default -n ${namespace} -o jsonpath='{.status.url}')
curl -k ${SERVICE_HOSTNAME}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

## Kubeflow training: tfJob

## Simple training

```shell
export namespace=raffa
oc apply -f ./training/tfjob/tfjob-minst.yaml -n ${namespace}
```

### tf gpu-distributed training

#### Create distributed training job

```shell
export namespace=raffa
oc apply -f ./training/tfjob/tfjob-gpu.yaml -n ${namespace}
```

## Kubeflow training: pytorch

### pytorch gpu-distributed training

```shell
docker build -t quay.io/raffaelespazzoli/pytorch-dist-mnist_test:1.0 -t quay.io/raffaelespazzoli/pytorch-dist-mnist_test:latest ./training/pyjob
docker push quay.io/raffaelespazzoli/pytorch-dist-mnist_test:1.0 quay.io/raffaelespazzoli/pytorch-dist-mnist_test:latest
export namespace=raffa
oc apply -f ./training/pyjob/pyjob-distributed-training.yaml -n ${namespace}
```

## Kubeflow pipelines

Kubeflow pipelines work in OpenShift with the [k8sapi](https://argoproj.github.io/argo-workflows/workflow-executors/#kubernetes-api-k8sapi) executor.
In order for it to work, steps that have output must write it to an emptyDir volume, as explained [here] (https://argoproj.github.io/argo-workflows/empty-dir/)

The out of the box example don't follow this pattern and will not work.

upload this file ./pipeline/control-structures.yaml into a new kubeflow pipeline from the dashboard to test a pipeline.

## remove kubeflow

```shell
./kustomize --load-restrictor=LoadRestrictionsNone build ./kubeflow1.3 | oc delete -f -
```

## Disable node autoscaling

```shell
oc patch ClusterAutoscaler default --type='json' -p='[{"op": "replace", "path": "/spec", "value":{}}]'
```
