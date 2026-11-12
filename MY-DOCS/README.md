# Quick Start

**The quickstart deploys a complete inference system: an intelligent router in front of two vLLM model servers.** It uses **Standalone Mode**, with its own Envoy proxy. [llm-d](https://llm-d.ai/docs/getting-started/quickstart?utm_source=chatgpt.com)

![LLM-D Arch](images/mermaid-diagram-8.png)

The gateway forwards the request; the EPP chooses the model server.

| Quickstart step | What it accomplishes |
|---|---|
| Clone the repository | Downloads the matching manifests and configuration |
| Source `guides/env.sh` | Loads chart locations and version settings |
| Set `GUIDE_NAME=quickstart` | Names this installation |
| Install GIE CRDs | Enables resources such as `InferencePool` |
| Create namespace and Hugging Face secret | Prepares the deployment location and model-download credentials |
| Install the router with Helm | Starts intelligent routing with Envoy |
| Apply model manifests with Kustomize | Starts two `Qwen/Qwen3-32B` replicas |
| Send a request | Tests inference through the router’s internal Service |

The test runs inside Kubernetes because the guide uses a **ClusterIP**, rather than a public endpoint. [llm-d](https://llm-d.ai/docs/getting-started/quickstart?utm_source=chatgpt.com)

**A subtle detail:** `GUIDE_NAME=quickstart` names the installation, while the commands explicitly load files from `guides/optimized-baseline/`. The installation name and recipe directory differ here. [llm-d](https://llm-d.ai/docs/getting-started/quickstart?utm_source=chatgpt.com)

For your L4 lab, retain this structure and replace the default model-server configuration with **two smaller-model replicas, each using one L4**. Keep the request’s model name consistent with the deployed model. That lets you explore routing between independent replicas before adding multi-node inference or a GKE-managed Gateway.

# Installation Instructions

## 1. Verify Gateway API Inference Extension resources

```bash
kubectl api-resources --api-group=inference.networking.k8s.io
NAME             SHORTNAMES   APIVERSION                       NAMESPACED   KIND
inferencepools   infpool      inference.networking.k8s.io/v1   true         InferencePool
```

## 2. Deploy the llm-d Router (Standalone Mode)

The llm-d Router provides the intelligent load balancing. In Standalone Mode, it includes a built-in proxy (Envoy). Apply smaller lab settings

```bash
helm upgrade --install "${GUIDE_NAME}" \
  "${ROUTER_STANDALONE_CHART}" \
  -f guides/recipes/router/base.values.yaml \
  -f guides/optimized-baseline/router/optimized-baseline.values.yaml \
  --set router.epp.resources.requests.cpu=500m \
  --set router.epp.resources.requests.memory=512Mi \
  --set router.epp.resources.limits.cpu=1 \
  --set router.epp.resources.limits.memory=1Gi \
  --set router.proxy.resources.requests.cpu=250m \
  --set router.proxy.resources.requests.memory=256Mi \
  --set router.proxy.resources.limits.cpu=1 \
  --set router.proxy.resources.limits.memory=512Mi \
  -n "${NAMESPACE}" \
  --version "${ROUTER_CHART_VERSION}"
```

These chart keys configure the EPP and proxy separately. GitHub The proposed settings request 750m CPU and 768 MiB memory combined—a starting point for light lab traffic, not production sizing.




## 3. Verify the new Pods

```bash
kubectl -n "${NAMESPACE}" rollout status deployment/quickstart-epp \
  --timeout=180s

kubectl -n "${NAMESPACE}" get pods
kubectl -n "${NAMESPACE}" get events --sort-by=.lastTimestamp
```

This deploys only the router on the CPU node. Gemma/vLLM model replicas still need the L4 nodes. 

## 4. Verify the InferencePool and router Service

```bash
kubectl -n llm-d-quickstart get inferencepool
kubectl -n llm-d-quickstart get svc quickstart-epp
```

## 5. Inspect the pool’s model-pod selector and target port

```bash
kubectl -n llm-d-quickstart get inferencepool quickstart -o yaml
```

```yaml
spec:
  selector:
    matchLabels:
      # Model-serving pods must carry these labels.
  targetPorts:
    - number: 8000
```

## 6. Create the overlays for gemma3

```bash
mkdir -p guides/optimized-baseline/modelserver/gpu/vllm/gemma-l4

cat > guides/optimized-baseline/modelserver/gpu/vllm/gemma-l4/kustomization.yaml <<'YAML'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../base

patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
    patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: modelserver
      spec:
        replicas: 2
        template:
          spec:
            nodeSelector:
              cloud.google.com/gke-nodepool: l4-pool
            tolerations:
              - key: nvidia.com/gpu
                operator: Exists
                effect: NoSchedule
            affinity:
              podAntiAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  - labelSelector:
                      matchLabels:
                        llm-d.ai/guide: optimized-baseline
                    topologyKey: kubernetes.io/hostname
            containers:
              - name: modelserver
                args:
                  - google/gemma-3-1b-it
                  - --host=0.0.0.0
                  - --port=8000
                  - --tensor-parallel-size=1
                  - --max-model-len=4096
                  - --gpu-memory-utilization=0.85
                  - --enable-prefix-caching
                  - --shutdown-timeout=45
                resources:
                  requests:
                    cpu: "2"
                    memory: 8Gi
                    nvidia.com/gpu: 1
                  limits:
                    cpu: "4"
                    memory: 16Gi
                    nvidia.com/gpu: 1
            volumes:
              - name: shm
                emptyDir:
                  medium: Memory
                  sizeLimit: 2Gi
```

## 7. Apply the gemma3

```bash
kubectl apply -n "$NAMESPACE" \
  -k guides/optimized-baseline/modelserver/gpu/vllm/gemma-l4/
```

## 8. Apply the qwen-l4

```bash
kubectl apply -n "$NAMESPACE" \
  -k guides/optimized-baseline/modelserver/gpu/vllm/qwen-l4/
```