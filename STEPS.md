# CKS GTC Demo

## Installation

### Setup your service account token

- Add pull-secret

```bash
mkdir -p ~/.config/containers
cp ~/token.json ~/.config/containers/auth.json
```

### Install

> Make sure your $KUBECONFIG is set to an absolute path!

```bash
# deploy pre-requisites
make deploy-all
```

```bash
# confirm install working
make status

=== Deployment Status ===
cert-manager-operator:
NAME                                                       READY   STATUS    RESTARTS   AGE
cert-manager-operator-controller-manager-5f95bc4bd-qtshq   1/1     Running   0          8m29s

cert-manager:
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-699cdbb7db-zfkjz              1/1     Running   0          8m27s
cert-manager-cainjector-7f876dc699-8n44x   1/1     Running   0          8m27s
cert-manager-webhook-6487d5b898-7cwzr      1/1     Running   0          8m27s

istio:
NAME                                    READY   STATUS    RESTARTS   AGE
servicemesh-operator3-9cf6695cd-8qdzc   1/1     Running   0          8m16s

lws-operator:
NAME                                      READY   STATUS    RESTARTS   AGE
lws-controller-manager-58c5dd85bf-kd469   1/1     Running   0          6m21s
lws-controller-manager-58c5dd85bf-zqdfj   1/1     Running   0          6m21s
openshift-lws-operator-89f74cd5b-k2nlf    1/1     Running   0          7m54s

kserve:
NAME                                        READY   STATUS    RESTARTS   AGE
kserve-controller-manager-5c69c4cfc-jjms2   1/1     Running   0          6m44s

kserve config:
NAME                                             AGE
kserve-config-llm-decode-template                6m44s
kserve-config-llm-decode-worker-data-parallel    6m44s
kserve-config-llm-prefill-template               6m44s
kserve-config-llm-prefill-worker-data-parallel   6m44s
kserve-config-llm-router-route                   6m44s
kserve-config-llm-scheduler                      6m44s
kserve-config-llm-template                       6m44s
kserve-config-llm-template-amd-rocm              6m44s
kserve-config-llm-worker-data-parallel           6m44s

=== Readiness Checks ===
-n cert-manager webhook: 
Ready

=== API Versions ===
-n InferencePool API: 
v1 (inference.networking.k8s.io)
-n Istio version: 
v1.27-latest
```

```bash
# deploy the gateway
./scripts/setup-gateway.sh
```


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


## Intelligent Inference Scheduling

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
kubectl apply -f intelligent-inference-scheduling/qwen-pvc.yaml
kubectl apply -f intelligent-inference-scheduling/download-job.yaml
```

- deploy
```bash
kubectl apply -f intelligent-inference-scheduling/deploy.yaml
```

### Make Inference Request

- port forward (in separate terminal)
```bash
kubectl port-forward svc/inference-gateway-istio 8080:80 -n opendatahub
```

- curl the endpoint
```bash
curl http://localhost:8080/llm-d-rhaii/qwen/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-32B",
    "messages": [
      {"role": "user", "content": "Hello!"}
    ],
    "max_tokens": 50
  }'
```

### Run Benchmark

