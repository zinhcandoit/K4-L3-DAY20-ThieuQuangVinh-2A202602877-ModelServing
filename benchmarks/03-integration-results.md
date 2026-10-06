# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3572.7 | 3572.8 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 1870.0 | 1870.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 3736.7 | 3736.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **3059.8** · total **3059.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it accounts for **SLOs (Service Level Objectives)**.

The context explicitly states:
> "Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets."

In contrast, **Raw Throughput** ignores SLOs. Therefore, Goodput is useful because it ensures that the system's performance is measured agai

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing the Key-Value (KV) cache in non-contiguous pages.

By using non-contiguous pages, the model avoids wasting most of the GPU's memory on the unused parts of the cache, thereby improving memory efficiency.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill and decode are bound to different memory bandwidth or compute resources**, allowing the system to avoid bottlenecks.

Based on the context:
*   **Prefill** is described as **compute-bound**, meaning it requires significant processing power.
*   **Decode** is described as **memory-bandwidth-bound**, meaning it requires significant data transfer spee


## Which N16-N19 pieces are real

- **Phân loại các thành phần N16–N19:**
  - **N16 Cloud/IaC:** **stub** (chạy hoàn toàn local trên môi trường máy cá nhân / WSL2, không kết nối hạ tầng cloud).
  - **N17 Data pipeline:** **stub** (dùng tập dữ liệu mẫu in-memory `TOY_DOCS`, không có pipeline ETL).
  - **N18 Lakehouse:** **stub** (không kết nối Lakehouse hay Delta/Iceberg table).
  - **N19 Vector + features:** **stub** (sử dụng thuật toán keyword overlap fallback trực tiếp trong script, không dùng embedding server hay Vector DB thật).
  - *(N20 Model Serving: **real** — phục vụ bằng `llama-server` thật qua endpoint `/v1/chat/completions`).*

- **Stage chiếm thời gian nhiều nhất:**
  - **LLM stage** chiếm áp đảo với trung bình **3059.8 ms** (chiếm **100%** tổng latency 3059.9 ms; embed và retrieve đều tốn 0.0 ms do chỉ tính toán trùng khớp từ khóa in-memory). Con số này hoàn toàn đúng như kỳ vọng đối với một pipeline stubbed retrieval.

- **Chiến lược giảm 2 lần latency (Halve latency):**
  - Áp dụng **Định luật Amdahl**, bắt buộc phải tấn công trực tiếp vào **LLM stage** vì nó chiếm trọn 100% thời gian thực thi:
    1. **Bật GPU Offload (`-ngl 99`):** Đẩy model lên GPU RTX 3070 thay vì chạy thuần CPU (`ngl: 0`). Băng thông VRAM (GDDR6) vượt trội sẽ đẩy tốc độ decode từ ~47 tok/s lên >150 tok/s, lập tức giảm latency toàn pipeline xuống dưới 1s (giảm >3×).
    2. **Kiểm soát Context Budget & Siết `max_tokens`:** Rút gọn context chunk nạp vào prompt (giảm chi phí prefill / TTFT) và giới hạn `max_tokens` ngắn gọn hơn thay vì sinh dài đến 200 tokens.
