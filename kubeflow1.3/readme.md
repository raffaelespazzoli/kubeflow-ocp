# Kubeflow 1.3

## clone the manifest repo

./kustomize --load-restrictor=LoadRestrictionsNone build ./kubeflow1.3 | oc apply -f -