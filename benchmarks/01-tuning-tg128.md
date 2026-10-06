# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 10.9 | 65% |
| 2 | 15.5 | 93% |
| 4 | 16.7 | 100% |
| 8 | 12.5 | 75% |
| 16 | 12.2 | 73% |

**Best**: `-t 4` at 16.7 tok/s
**Slowest tested**: `-t 1` at 10.9 tok/s (1.53x spread)
**Against the physical-core default** (`-t 4`, 16.7 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

The knee is at 4 threads, exactly the 4 physical cores: throughput peaks at 16.7
tok/s, then drops to 12.5 and 12.2 tok/s at 8 and 16 threads. Decode is
memory-bandwidth-bound, so SMT and oversubscription add scheduling/cache contention
without adding memory bandwidth. The measured 4-thread setting is therefore the
best default on this laptop.
