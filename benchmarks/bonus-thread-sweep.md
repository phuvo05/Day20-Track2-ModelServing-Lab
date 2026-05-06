# Bonus — Thread sweep

Model: `tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf`  ·  GPU layers: `0` (CPU-only build)

| threads | tg128 (tok/s) |
|---:|---:|
| 1 | 13.2 |
| 2 | 16.9 |
| 6 | 18.8 |
| 12 | 18.7 |
| 24 | 16.9 |

**Best**: `-t 6` at 18.8 tok/s (build: d5003b6e4)

The curve peaks at 6 threads (half of physical cores), then drops slightly at 12 and sharply at 24. This is textbook memory-bandwidth saturation: extra threads fight over the same memory channels and slow each other down. The i5-12450H has 12P/12E cores on a laptop with shared memory — going beyond the number of physical cores (6 efficiency + 6 performance) yields diminishing returns because the work is memory-bandwidth-bound, not compute-bound.
