# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.93 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 8291 |

Highest sampled value was **3.93 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

- **Peak batch width:** Đạt **3.93 / 4 slots (98%)**, tiệm cận tối đa `--parallel 4`. Điều này chứng minh scheduler của `llama-server` đã thực sự kích hoạt **continuous batching**, gộp song song các request vào chung một bước decode thay vì phục vụ tuần tự (nếu tuần tự thì peak ≈ 1).
- **So sánh với Effective Concurrency (33.3 trong `02-server-results.md`):** Hai con số **không khớp nhau** vì phản ánh hai phạm vi đo khác nhau:
  - `n_busy_slots_per_decode` (3.93) đo **true slot utilization** bên trong compute engine: số slot thực sự tham gia decode đồng thời (bị giới hạn trên bởi `--parallel 4`).
  - `Effective concurrency` (33.3 từ Little's Law) đo **system occupancy**: tổng số request đang nằm trong toàn bộ hệ thống, bao gồm 4 request đang tính toán (`requests_processing = 4`) và 46 request đang xếp hàng chờ slot (`requests_deferred = 46`).
- **Độ tin cậy:** Cả hai đều chính xác cho mục đích riêng: tin `n_busy_slots_per_decode` từ `/metrics` khi đánh giá **năng lực gom batch và độ bận của compute engine**, và tin `effective concurrency` từ client khi đánh giá **mức độ bão hòa tải và áp lực hàng đợi (queue time)**.
