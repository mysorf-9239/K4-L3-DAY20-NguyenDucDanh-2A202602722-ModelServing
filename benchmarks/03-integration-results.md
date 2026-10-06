# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3609.6 | 3609.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2804.3 | 2804.4 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2717.7 | 2717.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **3043.9** · total **3044.0**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

All four are stubs. N16 (cloud/IaC), N17 (data pipeline), N18 (lakehouse) and N19 (vector index and features) were not wired in, so retrieval is the keyword overlap over the built-in TOY_DOCS and the embed stage is 0.0 ms. Only N20, the llama-server, is real.

The dominant stage is llm with 100% of total (3044 ms mean, against 0.0 ms for embed and retrieve), which is what I expected given there is no embedding model and the corpus has only a few documents. If I had to halve the latency I would attack the llm stage, specifically decode, since each answer needs 23 to 30 tokens at about 12 tok/s (roughly 1.9 to 2.6 s of the 2.7 to 3.6 s). Shorter answers (lower max tokens) would help most. Prefill is the second part at about 0.8 to 1.0 s for 110 to 150 tokens, and it would shrink with prompt caching of the fixed system prompt.
