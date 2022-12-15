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

Enable Useful Tekton Features
```
kubectl -n tekton-pipelines patch cm feature-flags \
    -p '{"data":{"enable-tekton-oci-bundles":"true"}}'
```

Install Chains

```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml
```

Configure Chains

```
kubectl -n tekton-chains patch cm chains-config -p '{"data": {
  "artifacts.oci.storage": "oci",
  "artifacts.pipelinerun.format": "in-toto",
  "artifacts.pipelinerun.storage": "oci",
  "artifacts.taskrun.format": "in-toto",
  "artifacts.taskrun.storage": "oci"
}}'
```

Generate Signing Secret
```
cosign generate-key-pair k8s://tekton-chains/signing-secrets
```

## Namespace Setup

This section helps setup a new namespace which contains the same registry authentication
found on your local system.

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
