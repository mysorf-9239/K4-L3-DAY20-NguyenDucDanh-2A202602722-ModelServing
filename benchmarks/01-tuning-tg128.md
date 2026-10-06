# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **8 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 15.6 | 100% |
| 4 | 14.4 | 92% |
| 8 | 12.8 | 82% |
| 16 | 10.7 | 68% |

**Best**: `-t 1` at 15.6 tok/s
**Slowest tested**: `-t 16` at 10.7 tok/s (1.46x spread)
**Against the physical-core default** (`-t 8`, 12.8 tok/s): 1.21x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

The curve does not look like the textbook shape. There is no knee near the 8 physical cores. The best result is at 1 thread (15.6 tok/s), and throughput only goes down as I add threads: 4 threads gives 92%, 8 gives 82% and 16 gives 68% of the best. So the default of `-t 8` was already 1.21x slower than the best setting.

My explanation is that the run uses `-ngl 99`, so all layers sit on the Apple GPU through Metal and the matrix multiplies for decode never touch the CPU threads. The CPU threads only do orchestration, and extra threads spin and synchronise around the graph without adding any useful work. On top of that, the CPU and the GPU share the same unified memory on the M2, so more CPU threads polling and touching memory take bandwidth away from the GPU, which is the thing that actually limits decode. The 16 thread point, which is 2x oversubscribed on 8 cores, loses the most, in line with that. In short the knob that matters on this machine is whether layers are offloaded, not how many CPU threads exist.
