Test this in two passes. First, verify that `quickstart-epp` can route requests to **both Qwen pods**. Then send a long shared prefix and check whether the picker favors the warm pod and whether vLLM actually reuses cached tokens.

### 1. Verify the pool and open the metrics endpoints

```bash
NS=llm-d-quickstart

kubectl -n "$NS" get inferencepool quickstart -o yaml
kubectl -n "$NS" get pods -o wide --show-labels
```

Confirm that the `InferencePool` selector matches **both Ready Qwen pods**, that its model-server port is `8000`, and that its picker reference points to `quickstart-epp`. If your overlay inherited `llm-d.ai/guide: optimized-baseline`, compare that value with the pool selector. A Ready pod outside the selector is outside the picker’s candidate set. [llm-d](https://llm-d.ai/docs/dev/architecture/core/inferencepool/?utm_source=chatgpt.com)

Run these in **three separate terminals**, substituting the two Qwen pod names:

```bash
# Terminal 1 — Standalone router HTTP traffic and EPP metrics
kubectl -n llm-d-quickstart port-forward svc/quickstart-epp 8080:80 9090:9090

# Terminal 2 — Qwen pod A metrics
kubectl -n llm-d-quickstart port-forward pod/<qwen-pod-a> 18001:8000

# Terminal 3 — Qwen pod B metrics
kubectl -n llm-d-quickstart port-forward pod/<qwen-pod-b> 18002:8000
```

This assumes you’re using the Standalone `quickstart-epp` Service with its HTTP listener on port `80`. Send inference requests to **8080** so they go through the picker. Use **18001** and **18002** to read each vLLM pod’s metrics. [llm-d](https://llm-d.ai/docs/0.7/getting-started/quickstart?utm_source=chatgpt.com)

### 2. Verify routing to both pods

In a fourth terminal, define these two helpers. `snapshot` shows cumulative counters, so compare the values **before and after** each test. [llm-d](https://llm-d.ai/docs/0.10/operations/observability/promql?utm_source=chatgpt.com)

```bash
request() {
  jq -n --arg prompt "$1" \
    '{model:"Qwen/Qwen3-0.6B", prompt:$prompt, max_tokens:8, temperature:0}' |
    curl -fsS http://127.0.0.1:8080/v1/completions \
      -H 'Content-Type: application/json' --data-binary @- |
    jq -c '{id, model, usage}'
}

snapshot() {
  for port in 18001 18002; do
    printf '\n--- vLLM on port %s ---\n' "$port"
    curl -fsS "http://127.0.0.1:$port/metrics" |
      awk '
        /^vllm:request_success_total[{ ]/       { requests += $NF }
        /^vllm:prefix_cache_hits_total[{ ]/     { hits += $NF }
        /^vllm:prefix_cache_queries_total[{ ]/  { queries += $NF }
        END {
          printf "completed=%g  cached_tokens=%g  queried_tokens=%g\n",
                 requests, hits, queries
        }'
  done
}

request 'Reply with the word ready.'
snapshot

for i in 1 2 3 4 5 6; do
  request "Independent routing test $i. Write one short sentence." &
done
wait

snapshot
```

If **both `completed` counters increase**, both pods served traffic through the router. A `3/3` split is unnecessary. For picker-side confirmation, check whether your version exports successful scheduling attempts labeled by endpoint. With multiple EPP replicas, the Service port-forward exposes one replica’s metrics at a time. [llm-d](https://llm-d.ai/docs/0.10/operations/observability/promql?utm_source=chatgpt.com)

```bash
curl -fsS http://127.0.0.1:9090/metrics |
  rg '^llm_d_epp_scheduler_attempts_total'
```

### 3. Warm one pod and check for prefix reuse

The first request warms whichever pod receives it. The next five requests reuse the **exact same long prefix** and vary only the question at the end. [vLLM](https://docs.vllm.ai/en/v0.14.0/usage/metrics/?utm_source=chatgpt.com)

```bash
RUN_ID="$(date +%s)-$RANDOM"
PREFIX="$(jq -nr --arg id "$RUN_ID" '
  "Cache experiment \($id)\n" +
  ([range(1; 101) |
    "Record \(.): The platform routes inference requests to available model servers."
  ] | join("\n"))
')"

snapshot
request "$(printf '%s\n\nQuestion: What does the platform route?' "$PREFIX")"
snapshot

for i in 1 2 3 4 5; do
  request "$(printf '%s\n\nQuestion %s: What receives the requests?' "$PREFIX" "$i")"
done

snapshot
```

| What you see | What it tells you |
|---|---|
| The warm-up increments pod A’s `completed` counter | Pod A served the first request and had the opportunity to cache its prefix. |
| Most repeats go to pod A and its `cached_tokens` climbs | vLLM reused the prefix on pod A. The routing pattern is consistent with cache affinity. |
| Repeats go to both pods and each eventually reports hits | Each pod can warm its own cache. Check the EPP profile before attributing the distribution to cache-aware routing. |
| `queried_tokens` increases while `cached_tokens` stays flat | The serving pod looked up the prefix but reported no reuse. Check which pod received the repeats and confirm prefix caching is active. |

**To confirm the picker is using cache locality**, check its active scheduling profile for a plugin such as `prefix-cache-scorer` or `prefix-cache-affinity-filter`. If your build exports `llm_d_epp_prefix_predicted_cached_tokens_sum`, look for it to increase during the repeat phase. The EPP metric reports **predicted reuse**. The vLLM counters report **actual reuse**. Your `--enable-prefix-caching` flag enables the vLLM side of the test. [llm-d](https://llm-d.ai/docs/dev/architecture/router/configuration?utm_source=chatgpt.com)

```bash
kubectl -n llm-d-quickstart get cm -o yaml |
  rg -n -C 3 'prefix-cache-(scorer|affinity-filter|producer)'

curl -fsS http://127.0.0.1:9090/metrics |
  rg '^llm_d_epp_prefix_(predicted_cached_tokens|prompt_tokens)_sum'
```