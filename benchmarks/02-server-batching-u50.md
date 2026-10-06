# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 20 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.84 of 4 slots (96%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 2138 |

Highest sampled value was **3.84 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

The peak was 3.84 of 4 slots, so the scheduler really was packing requests together. `requests_processing` was 4 and `requests_deferred` reached 46, which means 46 users were waiting for a free slot while 4 were being decoded. It matches the saturation reading: the effective concurrency from Little's Law was 10.3, well above 4 slots, so the extra load is sitting in a queue. The two numbers measure different things (one is occupancy including the queue, the other is real slot utilisation), but both say the 4 slots are full. For utilisation I trust the server gauge, and for queue pressure I trust `deferred`, since locust only averages requests that have finished.
