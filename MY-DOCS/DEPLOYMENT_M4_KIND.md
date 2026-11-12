# llm-d in M4 kind

1. Create a local cluster and install the Endpoint Picker
Start Docker Desktop and give its Linux VM about 10 GiB RAM as a starting point. Stop any native vLLM Metal server while running this lab. The commands below explicitly target kind-llmd-m4 so they don’t touch your GKE cluster.

```bash
brew install kind kubectl helm jq

kind create cluster --name llmd-m4
export KCTX=kind-llmd-m4
export NS=llm-d-m4

git clone https://github.com/llm-d/llm-d.git
cd llm-d
export REPO_ROOT="$PWD"
source guides/env.sh

kubectl --context "$KCTX" apply -f \
  "https://github.com/kubernetes-sigs/gateway-api-inference-extension/${GAIE_URL}/v1-manifests.yaml"

kubectl --context "$KCTX" wait --for=condition=Established \
  crd/inferencepools.inference.networking.k8s.io --timeout=60s

helm --kube-context kind-llmd-m4 upgrade --install m4 \
  "$ROUTER_STANDALONE_CHART" \
  --version "$ROUTER_CHART_VERSION" \
  --namespace llm-d-m4 --create-namespace \
  -f guides/recipes/router/base.values.yaml \
  -f guides/optimized-baseline/router/optimized-baseline.values.yaml \
  --set router.inferencePool.failureMode=FailClose \
  --set router.epp.flags.v=2 \
  --set-string router.proxy.resources.requests.cpu=250m \
  --set-string router.proxy.resources.requests.memory=256Mi \
  --set-string router.epp.resources.requests.cpu=500m \
  --set-string router.epp.resources.requests.memory=512Mi
```

The chart creates the InferencePool and the EPP. FailClose makes an EPP processing failure visible to the client. This test uses the InferencePool selector and requires no InferenceObjective resource.  

These are starting requests for your low-traffic lab. Together they reserve 750m CPU and 768 MiB, leaving room for the two Qwen replicas.

```bash
kubectl --context kind-llmd-m4 -n llm-d-m4 \
  rollout status deployment/m4-epp --timeout=180s

kubectl --context kind-llmd-m4 -n llm-d-m4 get pods
```

Once the new m4-epp pod shows 2/2 Running, deploy qwen-small. If the rollout times out, describe the new pod and check its events for the current blocker.

2. Deploy two small vLLM replicas
The pod label llm-d.ai/guide: optimized-baseline matches the pool selector in the llm-d values file. The containers use CPU inference with a 512-token context and a 1 GiB KV cache per replica. These are starting settings for your 16 GB machine.

```bash
kubectl --context "$KCTX" -n "$NS" apply -f - <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qwen-small
spec:
  replicas: 2
  selector:
    matchLabels:
      app: qwen-small
  template:
    metadata:
      labels:
        app: qwen-small
        llm-d.ai/guide: optimized-baseline
    spec:
      containers:
        - name: vllm
          image: vllm/vllm-openai-cpu:v0.33.0-arm64
          args:
            - Qwen/Qwen3-0.6B
            - --host=0.0.0.0
            - --port=8000
            - --dtype=float32
            - --max-model-len=512
            - --max-num-seqs=2
            - --enforce-eager
          env:
            - name: VLLM_CPU_KVCACHE_SPACE
              value: "1"
            - name: VLLM_CPU_OMP_THREADS_BIND
              value: nobind
            - name: OMP_NUM_THREADS
              value: "2"
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            periodSeconds: 5
          volumeMounts:
            - name: shm
              mountPath: /dev/shm
      volumes:
        - name: shm
          emptyDir:
            medium: Memory
            sizeLimit: 512Mi
YAML

kubectl --context "$KCTX" -n "$NS" rollout status \
  deployment/qwen-small --timeout=10m

kubectl --context "$KCTX" -n "$NS" get inferencepool -o yaml
kubectl --context "$KCTX" -n "$NS" get pods -l app=qwen-small -o wide
```