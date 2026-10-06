# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 33 | 0.55 | 16000 | 23000 | 25000 | 8.7 | 0.0% |
| 50 | 23 | 0.38 | 32000 | 55000 | 60000 | 12.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.69x** (14% of linear) |
| P95 latency | **2.39x** |
| Effective concurrency at 50 users | 12.4 vs `--parallel 4` slots (occupancy/slot ratio 3.09) |

**Saturated.** Throughput delivered only 0.69x for 5x the offered load, and effective concurrency (12.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.69x while P95 moved 2.39x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server is clearly saturated by 50 users: offered load increased 5x, but delivered
throughput fell to 0.69x (0.55 to 0.38 RPS), while P95 rose 2.39x (23 s to 55 s).
Effective concurrency was 12.4 versus four slots, and the server independently measured
3.85/4 busy slots with 46 deferred requests, so the extra latency is queue time. With a
30 s SLO, at least 95% of the 10-user requests meet it, but fewer than half of the
50-user requests do because their median is 32 s. I would first reduce output/context
token budgets, which lowers service time and queue depth directly; adding slots would
increase contention on the same four CPU cores.
