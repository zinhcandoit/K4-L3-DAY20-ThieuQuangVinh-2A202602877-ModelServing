# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 61 | 1.06 | 8300 | 12000 | 13000 | 8.6 | 0.0% |
| 50 | 75 | 1.28 | 28000 | 41000 | 42000 | 33.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.20x** (24% of linear) |
| P95 latency | **3.42x** |
| Effective concurrency at 50 users | 33.3 vs `--parallel 4` slots (occupancy/slot ratio 8.32) |

**Saturated.** Throughput delivered only 1.20x for 5x the offered load, and effective concurrency (33.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.20x while P95 moved 3.42x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

- **Điểm bão hòa & Bằng chứng:** Server đã bão hòa hoàn toàn tại mức 50 users (thực tế điểm bão hòa xuất hiện từ ngưỡng ~10–15 users). Bằng chứng rõ nhất là khi offered load tăng 5×, RPS thực tế chỉ tăng vỏn vẹn **1.20×** (1.06 lên 1.28 RPS, chỉ đạt 24% mức tăng tuyến tính), trong khi độ trễ P95 bùng nổ gấp **3.42×** (tăng từ 12s lên 41s).
- **Phân tách Queue time vs Compute time:** Phần latency tăng thêm hoàn toàn là **queue time**. Effective concurrency đạt **33.3** (gấp 8.32× số slot vật lý `--parallel 4`), kết hợp với metrics `requests_processing = 4` và `requests_deferred = 46` chứng minh thời gian tính toán thực tế (compute time) trên mỗi slot không đổi nhiều, nhưng các request phải xếp hàng chờ rất lâu để nhận được slot rảnh.
- **Knob đổi trước để nâng goodput@SLO:** 
  - Với SLO giả định P95 ≤ 15s, ở 50 users goodput@SLO hiện gần như bằng 0.
  - Knob đổi đầu tiên là **`-ngl` (bật GPU offload lên RTX 3070)**: Tận dụng băng thông VRAM vượt trội để tăng tốc độ decode, rút ngắn mạnh mẽ service time trên mỗi request, từ đó giải phóng slot nhanh chóng và triệt tiêu hàng đợi. (Nếu bắt buộc pure CPU, knob cần áp dụng là **Admission Control / Queue Timeout** ở reverse proxy để từ chối các request quá tải thay vì tích tụ queue làm vỡ SLO hàng loạt).
