# Bonus Challenge C7 — CPU Instruction Set Archaeology

**Hardware:** Intel Core i5-12450H (12th Gen) — 6P+6E cores, DDR5, laptop TDP
**OS:** Windows 11 via MSVC 19.50
**Model:** tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf

## Setup

Two builds of llama.cpp (d5003b6e4) compiled from source:

| Build | CMake flags | CPU instructions used |
|---|---|---|
| `llama-bench.exe` (optimized) | `-DGGML_NATIVE=ON` | **AVX2, FMA, F16C, BMI2** |
| `llama-bench-scalar.exe` | `-DGGML_NATIVE=OFF -DGGML_AVX2=OFF -DGGML_FMA=OFF -DGGML_F16C=OFF -DGGML_BMI2=OFF` | **AVX** (forced by MSVC `/arch:AVX` on x64) |

Even with all SIMD extensions explicitly disabled, MSVC on x64 enforces a minimum of `/arch:AVX` — so the "scalar" build still has 128-bit SIMD. The real baseline would require GCC with `-march=x86-64` (no SIMD), but MSVC won't produce that binary.

## Numbers (tg128, 3 runs, t=6)

| Build | tok/s | Δ vs scalar |
|---|---|---|
| Scalar (AVX baseline) | 7.36 | — |
| Optimized (AVX2+FMA+F16C) | 8.68 | +18% |

## What the result tells us

The 1.18× speedup from enabling AVX2/FMA/F16C is modest compared to what the slides suggest — and that's the key insight. The theoretical gain from 128-bit AVX → 256-bit AVX2 should be closer to 2× for floating-point matmuls, but:

1. **Q4_K quantization eliminates most FP ops.** The model's weights are 4-bit integers. The expensive SIMD kernels (FP matmul, softmax, layer-norm) operate on dequantized int8 → FP32 within each inner loop. The quantization overhead partially negates the SIMD gain.

2. **Memory bandwidth is the real bottleneck.** For a 1.1B-parameter model on a laptop with shared DDR5, the matrix multiplications spend more time waiting for data than doing compute. Even a 2× faster compute unit hits the memory wall first.

3. **The Intel i5-12450H is a hybrid P+E chip.** llama.cpp's default thread scheduler doesn't distinguish P-cores from E-cores. On a laptop, E-cores (smaller caches, lower clocks) running at full load cause thermal throttling that limits AVX2's advantage. A desktop Core i7 with all-P-cores would show a larger gap.

This is the same decision cloud providers face when choosing FA3 (Hopper) vs FA4 (Blackwell) vs FlashInfer (A100): the kernel compiler flag (`-DGGML_NATIVE=ON`) is the on-premise equivalent of picking a specific CUDA architecture. You only get the advertised speedup if you're not also memory-bandwidth-bound.
