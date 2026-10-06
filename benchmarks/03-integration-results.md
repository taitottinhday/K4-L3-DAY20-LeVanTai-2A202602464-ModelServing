# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.4 | 12768.4 | 12768.8 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 6128.6 | 6128.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 9935.5 | 9935.7 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **9610.8** · total **9611.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput is more useful than raw throughput** because it explicitly addresses the issue of **memory saturation**.

Here is the breakdown based on the text:

1.  **The Limitation of Raw Throughput**: The context states, "Throughput at saturation ignores SLOs." This means that if a system hits its maximum capacity (saturation), the raw throughput metric will stop incr

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing key-value pairs (KV cache) in non-contiguous pages.

By storing these pages in non-contiguous memory regions, the model avoids the wasted space that would otherwise be consumed by internal fragmentation, thereby optimizing the use of GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**.

This is because the context explicitly states that prefill is compute-bound and decode is memory-bandwidth-bound. By splitting them, the system can utilize different processing paths for each phase:
1.  **Prefill**: Can be processed efficiently using compute-bound resources (e.g., GPU or specialized h


## Which N16-N19 pieces are real

- N16 Cloud/IaC: stubbed (`localhost` only).
- N17 Data pipeline: stubbed (in-memory `TOY_DOCS`).
- N18 Lakehouse: stubbed (toy dictionary).
- N19 Vector/features: stubbed (keyword-overlap retrieval).
- N20 serving: real `llama-server`.

The LLM dominated at 9610.8/9611.1 ms (about 100%), which was expected; retrieval was
only 0.2 ms. To halve latency I would reduce output tokens or use the faster Q2 model
before optimizing the retrieval stub.
