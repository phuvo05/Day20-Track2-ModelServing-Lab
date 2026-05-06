# 02 — Server Load Test Results

Model: `tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf`
Server: `llama-cpp-python` via `llama_cpp.server`, 12 threads, 99 GPU layers, ctx=2048

## Locust 10 users, 60s

```
Type     Name    # reqs  # fails |  Avg (ms)  Min  Max    Med   req/s
POST     short       7   0 (0%)  |   25703   4044 48744  25000   0.12
```

Percentiles: P50=25000ms, P75=40000ms, P95=49000ms, P99=49000ms

## Locust 50 users, 60s

```
Type     Name    # reqs  # fails |  Avg (ms)  Min    Max      Med   req/s
POST     short       4   0 (0%)  |   35245  19803  51356  40000   0.07
```

Percentiles: P50=40000ms, P75=51000ms, P95=51000ms, P99=51000ms

## Notes

- /metrics endpoint is not available in the default llama-cpp-python build (would require custom `--metrics` build).
- KV-cache observation: at concurrency 50, queuing pressure increases median E2E from ~25s to ~40s, indicating the CPU-side inference pipeline saturates under concurrent requests.
- Running on WSL2 with RTX 4050 GPU passthrough; model loaded via llama-cpp-python CPU backend (CUDA not available in WSL due to missing CUDA Toolkit packages — network download too slow).
