# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 16 | 0.30 | 23000 | 41000 | 41000 | 6.6 | 0.0% |
| 50 | 21 | 0.38 | 23000 | 54000 | 55000 | 10.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.27x** (25% of linear) |
| P95 latency | **1.32x** |
| Effective concurrency at 50 users | 10.3 vs `--parallel 4` slots (occupancy/slot ratio 2.57) |

**Saturated.** Throughput delivered only 1.27x for 5x the offered load, and effective concurrency (10.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.27x while P95 moved 1.32x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 16 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

The server is already saturated at 10 users. Going from 10 to 50 users (5x offered load) raised RPS only from 0.30 to 0.38, which is 1.27x or 25% of linear, while P95 grew from 41 s to 54 s (1.32x). The numbers that convinced me are the effective concurrency of 10.3 against 4 slots and the 46 deferred requests seen in `/metrics`. Throughput is flat because all 4 slots were busy (3.84 of 4) and every extra user just waits. So the added latency is queue time, not compute time. Per-request compute is also slow on its own: about 12 tok/s per stream and a median of 23 s even at 10 users.

Both runs only completed 16 to 22 requests, so the percentiles are rough. For an SLO I would say P95 under 30 s. At 10 users P95 was already 41 s, so I keep close to none of the goodput at that SLO, and even 10 users is too much for this laptop. Before touching anything I would lower the load or the output length, because the cost per request is high. The first knob I would turn is `--parallel`. More slots might raise aggregate tokens per second a bit since batching shares the weight reads, but on a small laptop GPU each stream gets slower, so I would test 2, 4 and 8 and compare RPS and P95 together instead of guessing. I did not run that comparison here.
