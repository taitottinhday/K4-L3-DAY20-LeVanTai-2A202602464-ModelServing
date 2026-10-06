# Reflection - Day 20 Lab (Personal Report)

**Họ Tên:** Lê Văn Tài
**MSSV:** 2A202602464
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 - 10 điểm)*

- **OS:** Windows 10 (AMD64)
- **CPU:** Intel(R) Core(TM) i5-10300H CPU @ 2.50GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** Probe Windows không báo extension riêng
- **RAM:** 7.8 GB
- **Accelerator:** NVIDIA GeForce GTX 1650 (4096 MiB), CUDA/Vulkan detected
- **llama.cpp asset đã tải:** prebuilt Windows CUDA runtime b10488
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop local.

**Setup story:** RAM 7.8 GB nên chọn Qwen nhỏ. Bootstrap ban đầu gọi nhầm pip hệ thống,
khiến `huggingface_hub` không nằm trong `.venv`; tôi sửa bằng `.\.venv\Scripts\python.exe
-m pip`, sau đó runtime và cả hai weights tải thành công.

---

## 2. Đo lường  *(rubric 3, 4, 5 - 20 điểm)*

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 4360 | 326 / 964 | 66.9 / 76.2 | 4175 / 5313 / 5313 | 14.9 |
| UD-Q2_K_XL | 0.39 | 2605 | 626 / 2252 | 86.7 / 199.4 | 7104 / 13429 / 13429 | 11.5 |

**Quan sát:** Q2 nhỏ hơn 0.11 GB (22%) nhưng decode chậm hơn 1.30x: 11.5 so với
14.9 tok/s; TTFT và E2E tail latency cũng cao hơn. Máy chạy CPU-only với 4 physical
cores nên chi phí dequantization của Q2 lớn hơn lợi ích giảm lượng byte đọc từ RAM.
Vì vậy Q4 là mặc định hợp lý hơn trên máy này: nhanh hơn và giữ nhiều precision hơn;
Q2 chỉ đáng dùng khi thực sự cần tiết kiệm thêm 0.11 GB RAM.

---

## 3. Serving under load  *(rubric 8, 9, 10 - 20 điểm)*

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.55 | 16000 | 23000 | 25000 | 8.7 | 0.0% |
| 50 | 0.38 | 32000 | 55000 | 60000 | 12.4 | 0.0% |

- **Offered load tăng 5x, throughput thực còn:** 0.69x
- **P95 tăng:** 2.39x
- **Effective concurrency ở 50 users:** 12.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`:** 3.85 / 4 slots (96%)

**Saturation reading:** 50 users đã vượt saturation: tải tăng 5x nhưng throughput giảm từ
0.55 xuống 0.38 RPS (0.69x), trong khi P95 tăng từ 23 s lên 55 s (2.39x). Busy slots đạt
3.85/4 và deferred đạt 46; effective concurrency 12.4 so với 4 slots xác nhận queue tích
lũy. Với SLO 30 s, ít nhất 95% request ở 10 users đạt SLO, nhưng dưới 50% ở 50 users đạt
vì median đã là 32 s. Tôi sẽ giảm output/context tokens trước khi tăng slots để giảm service
time và queue depth; thêm slots trên bốn CPU core chỉ làm tăng tranh chấp tài nguyên.

---

## 4. Integration  *(rubric 12, 13 - 15 điểm)*

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost only | stub |
| N17 Data pipeline | in-memory `TOY_DOCS` | stub |
| N18 Lakehouse | toy dictionary | stub |
| N19 Vector + features | keyword-overlap retrieval | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query):

- embed: 0.0 ms
- retrieve: 0.2 ms
- llm: 9610.8 ms
- **stage chiếm nhiều nhất:** llm (100% của total 9611.1 ms)

**Reflection:** Kết quả phù hợp kỳ vọng: keyword retrieval gần như miễn phí, còn LLM chiếm
toàn bộ latency. Nếu cần giảm pipeline 2x, tôi sẽ giảm số token sinh hoặc thử GPU offload;
tối ưu retrieval stub không tạo khác biệt đáng kể ở số đo hiện tại.

---

## 5. The single change that mattered most  *(rubric 11 - 10 điểm)*

**Change:** tăng thread count từ 1 lên 4, đúng số physical cores.

```
before:  10.9 tok/s (-t 1)
after:   16.7 tok/s (-t 4)
speedup: 1.53x
```

Đây là knee rõ nhất trong sweep: 1 -> 2 -> 4 threads tăng từ 10.9 lên 16.7 tok/s,
nhưng 8 và 16 threads giảm còn 12.5 và 12.2 tok/s. Decode phải đọc weights ở mỗi token,
nên bị giới hạn bởi memory bandwidth; sau 4 physical cores, SMT và oversubscription không
tạo thêm bandwidth mà làm các thread tranh chấp cache, memory channels và lịch CPU. Vì vậy
4 threads là cấu hình tốt nhất đo được trên chính laptop này, không phải chỉ là giả định
"càng nhiều thread càng nhanh".

---

## 6. Bonus (optional)

Chưa làm bonus; ưu tiên hoàn tất và verify đầy đủ base track 100 điểm.

---

## 7. Điều làm tôi ngạc nhiên nhất

Quantization nhỏ hơn không mặc định nhanh hơn: Q2 tiết kiệm 22% dung lượng nhưng lại
decode chậm hơn Q4 1.30x trên CPU này. Kết quả cho thấy phải đo trên đúng phần cứng,
vì chi phí dequantization có thể lớn hơn lợi ích memory bandwidth.

---

## 8. Self-check trước khi push

- [x] `hardware.json` generated
- [x] `models/active.json` generated
- [x] Baseline, tuning, serving, load, batching và integration reports generated
- [x] 5 required screenshots present
- [x] `.\lab.ps1 verify` exit 0
- [ ] Repo public và URL đã paste vào LMS
- [x] Không commit `models/*.gguf`, `runtime/` hay `.env`

---

## 9. Khai báo sử dụng AI

Đã dùng Codex để đọc rubric/Markdown, debug lỗi Windows PowerShell và virtualenv, rà soát
output, và hỗ trợ diễn đạt báo cáo. Tất cả số liệu trong báo cáo được chạy lại trên laptop
này; không dùng số liệu hoặc screenshot của người khác.
