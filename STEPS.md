# CKS GTC Demo

## Installation

### Setup your service account token

- Add pull-secret (from OpenShift table)

```bash
mkdir -p ~/.config/containers
cp ~/token.json ~/.config/containers/auth.json
```

### Install

> Make sure your $KUBECONFIG is set to an absolute path!

```bash
# deploy pre-requisites
make deploy-all

# deploy the gateway
./scripts/setup-gateway.sh

```bash
robertgshaw@Roberts-MacBook-Pro validation % kubectl get gateways -A
NAMESPACE     NAME                CLASS   ADDRESS     PROGRAMMED   AGE
opendatahub   inference-gateway   istio   10.16.4.1   True         18m
```

## Hello, World Deployment

### Setup

- create namespace
```bash
export NAMESPACE=llm-d-rhaii
kubectl create namespace $NAMESPACE
```

- copy access token + configure serviceaccount
```bash
kubectl get secret redhat-pull-secret -n istio-system -o json | \
  jq 'del(.metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp, .metadata.annotations, .metadata.labels) | .metadata.namespace = "'$NAMESPACE'"' | \
  kubectl create -f -

kubectl patch serviceaccount default -n $NAMESPACE \
  -p '{"imagePullSecrets": [{"name": "redhat-pull-secret"}]}'
```

### Download Model and Deploy

- download model to cluster
```bash
kubectl apply -f hello-world/gpt-oss-pvc.yaml
kubectl apply -f hello-world/download-job.yaml
```

- deploy
```bash
kubectl apply -f hello-world/deploy.yaml
```

### Make Inference Request

- port forward (in separate terminal)
```bash
kubectl port-forward svc/inference-gateway-istio 8080:80 -n opendatahub
```

- curl the endpoint
```bash
curl http://localhost:8080/llm-d-rhaii/gpt-oss/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-oss-120b",
    "messages": [
      {"role": "user", "content": "Hello!"}
    ],
    "max_tokens": 50
  }'
```

