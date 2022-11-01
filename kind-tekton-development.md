# Tekton Development with Kind

Notes on how to use Kind to develop on Tekton Chains.

## Setup

Start a new kind cluster. Skip if you already have one.

```
kind create cluster --image kindest/node:v1.24.6
```

Install Tekton

```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

Install Chains

```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml
```

Configure Chains

```
kubectl -n tekton-chains edit cm chains-config

# Add the following:
data:
  artifacts.oci.storage: oci
  artifacts.pipelinerun.format: in-toto
  artifacts.pipelinerun.storage: oci
  artifacts.taskrun.format: in-toto
  artifacts.taskrun.storage: oci
```

## Namespace Setup

Create a new namespace.

```
kubectl create namespace minimal-container
```

Upload image registry secret and link to default service account.

```
kubectl -n minimal-container create secret docker-registry $USER \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json
# Must either use oc command, or edit the ServiceAccount manually.
oc -n minimal-container secrets link default $USER --for=mount,pull
```
