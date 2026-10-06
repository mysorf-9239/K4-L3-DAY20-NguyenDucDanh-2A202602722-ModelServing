# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5112 | 344 / 689 | 81.9 / 94.7 | 5486 / 6431 / 6431 | 12.2 |
| UD-Q2_K_XL | 2.24 | 5095 | 364 / 567 | 69.9 / 79.0 | 4753 / 5355 / 5355 | 14.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.17x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

On this M2 the 2-bit file decodes about 1.17x faster than the 4-bit one (14.3 vs 12.2 tok/s, TPOT P50 69.9 vs 81.9 ms) and is 0.73 GB smaller (2.24 vs 2.97 GB). TTFT did not improve (364 vs 344 ms at P50), which makes sense because prefill is compute work and the smaller weights do not help it. These numbers come from the first `make bench` run, so the page cache was cold for the first quant and the very first requests may be a bit slow.

I asked both servers the same two questions. The explanation of memory-bandwidth-bound decoding came out about equally good on both. On "17 * 23, show the steps", the 4-bit model wrote 17 x 20 = 340 correctly, while the 2-bit model wrote 17 x 2 = 34 and dropped the place value, so its working was sloppier. A 17% speedup and 0.7 GB saved is not a lot, and I already have 16 GB of RAM, so on this machine I would not accept the quality loss. I would pick 2-bit only on a device where 2.97 GB really does not fit.
