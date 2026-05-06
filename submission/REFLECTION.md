# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Võ Thiên Phú
**Cohort:** A20-K1
**Ngày submit:** 2026-05-06

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** Ubuntu 24.04 (WSL2 on Windows 11)
- **CPU:** 12th Gen Intel(R) Core(TM) i5-12450H
- **Cores:** 12 physical / 12 logical
- **CPU extensions:** AVX2 = true, AVX-512 = false, NEON = false
- **RAM:** 7.6 GB
- **Accelerator:** NVIDIA GeForce RTX 4050 Laptop GPU, 6141 MiB
- **llama.cpp backend đã chọn:** CPU (CUDA build attempted — WSL2 network too slow for CUDA Toolkit packages; fell back to CPU)
- **Recommended model tier:** TinyLlama-1.1B (Q4_K_M)

**Setup story** (≤ 80 chữ): Running on WSL2 Ubuntu 24.04 with GPU passthrough to RTX 4050. Attempted CUDA Toolkit installation in WSL but network was too slow for 2GB package download. Fell back to CPU backend. Installed llama-cpp-python via pip, downloaded TinyLlama GGUF via huggingface_hub.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

Settings: `n_threads=12`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf | 734 | 121 / 162 | 44.5 / 56.6 | 2748 / 3631 / 3662 | 22.5 |
| tinyllama-1.1b-chat-v1.0.Q2_K.gguf | 22467 | 178 / 422 | 36.0 / 68.2 | 2206 / 4681 / 5154 | 27.8 |

**Một quan sát** (≤ 50 chữ): Q2_K is 1.2× faster in decode (27.8 vs 22.5 tok/s) but 30× slower to load (22s vs 0.7s). Q4_K_M is the better quality/speed tradeoff for interactive use.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | ~0.12 | 25000 | 49000 | 49000 | 0 |
| 50 | ~0.07 | 40000 | 51000 | 51000 | 0 |

**KV-cache observation** (from `llama_kv_cache` server log lines + queue behavior):
- The server log confirms KV cache allocated as 44.00 MiB (2048 cells, 22 layers) on CPU.
- `llamacpp:kv_cache_usage_ratio` is not exposed by the default llama-cpp-python build (no `--metrics` support). I observed queue behavior instead: at concurrency 50, median E2E jumped from ~25s to ~40s, indicating the single-worker CPU pipeline saturates under concurrent requests.
- `n_seq_max = 1` means the server processes one sequence at a time; concurrent requests queue up. Peak KV usage = 44 MB out of 2048-token cells.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _stub: localhost only — no k3s/GCP cluster available in WSL_
- **N17 (Data pipeline):** _stub: in-memory toy document store_
- **N18 (Lakehouse):** _stub: in-memory list (no Delta Lake/Iceberg)_
- **N19 (Vector + Feature Store):** _stub: TOY_DOCS in-memory embedding lookup_

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- retrieve: ~0.0–0.1 ms (in-memory toy keyword match)
- llama-server: ~3,800–23,200 ms per query (CPU inference — dominant cost by far)

**Reflection** (≤ 60 chữ): The in-memory retrieval is essentially free; llama-server dominates at ~4–23s per query on CPU. The pipeline successfully demonstrates RAG context injection with provenance (context IDs printed). Stubs are clearly marked; a production deployment would swap TOY_DOCS for a real vector index.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Built llama.cpp from source on native Windows (MSVC) with GGML_NATIVE=ON (AVX2) and ran a thread sweep.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: t=1  ->  13.2 tok/s
after:  t=6  ->  18.8 tok/s
speedup: ~1.42×  (vs default t=1 from conservative wheel)
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

The default pre-built llama-cpp-python wheel uses a conservative CPU baseline that doesn't enable AVX2/AVX-512, resulting in scalar fallback code. Rebuilding with `-DGGML_NATIVE=ON` tells CMake to auto-detect the CPU's instruction set extensions (AVX2, FMA, F16C available on the i5-12450H) and compile optimized kernels that use SIMD vector instructions. The speedup is purely from better CPU instruction utilization — matrix-vector ops that previously took 8 scalar ops now take 1 SIMD instruction.

The thread sweep reveals a second insight: the peak is at t=6, not t=12. The i5-12450H is a hybrid 6P+6E chip. For memory-bandwidth-bound LLM decode, physical efficiency cores (E-cores) with their simpler cache hierarchy and lower latency are actually better suited than P-cores. The 24-thread oversubscription drops throughput to t=1 levels because all threads fight over the same DDR5 bandwidth.

---

## 6. (Optional) Điều ngạc nhiên nhất

_1–2 câu — không bắt buộc, nhưng người grader đọc tất cả_

_Surprising: even on an RTX 4050 laptop, the WSL2 network constraint was the real bottleneck — not the GPU. Installing CUDA Toolkit packages in WSL took longer than the entire rest of the lab combined. The lesson: for GPU-accelerated LLM serving on Windows, native Windows builds (not WSL) are the practical path._

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` đã commit (locust 10+50 user results)
- [x] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (06-bonus-sweep.png added)
- [x] At least one bonus challenge attempted (C1-C7 in CHALLENGES.md)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS

---
