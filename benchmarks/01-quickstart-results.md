# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 4360 | 326 / 964 | 66.9 / 76.2 | 4175 / 5313 / 5313 | 14.9 |
| UD-Q2_K_XL | 0.39 | 2605 | 626 / 2252 | 86.7 / 199.4 | 7104 / 13429 / 13429 | 11.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.30x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

UD-Q2_K_XL is 0.11 GB (22%) smaller, but on this CPU-only run it decoded 1.30x
slower than Q4_K_M (11.5 versus 14.9 tok/s), with both TTFT and E2E tail latency
also worse. The smaller weights therefore do not justify the dequantization cost
on this 4-core machine. Q4_K_M is the better default here because it is faster and
also preserves more precision; Q2 is useful only when the extra 0.11 GB RAM saving
is required.
