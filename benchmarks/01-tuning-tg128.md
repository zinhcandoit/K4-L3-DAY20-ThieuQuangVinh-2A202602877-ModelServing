# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 18.2 | 37% |
| 4 | 45.4 | 91% |
| 8 | 49.9 | 100% |
| 16 | 21.2 | 42% |
| 32 | 2.7 | 5% |

**Best**: `-t 8` at 49.9 tok/s
**Slowest tested**: `-t 32` at 2.7 tok/s (18.49x spread)
**Against the physical-core default** (`-t 8`, 49.9 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

- **Vị trí của Knee:** Điểm uốn (knee) và đỉnh hiệu năng nằm chính xác tại **`-t 8`** (49.9 tok/s), trùng khớp với số **physical cores** của CPU (8 physical cores).
- **Cơ chế tại sao knee ở đó:**
  - **Giới hạn băng thông (Memory Bandwidth-bound):** Giai đoạn decode (`tg128`) bị nghẽn ở băng thông bộ nhớ vì mỗi token sinh ra đều phải nạp toàn bộ trọng số mô hình từ RAM/cache vào thanh ghi. Từ 1 -> 8 threads, thông lượng tăng từ 18.2 lên 49.9 tok/s và tiệm cận trần băng thông bộ nhớ phần cứng.
  - **Tranh chấp tài nguyên & Overhead khi vượt quá physical cores:** Khi tăng lên `-t 16` (logical cores), hiệu năng giảm sâu xuống 21.2 tok/s (chỉ còn 42% vs best), và chạm đáy ở `-t 32` với 2.7 tok/s (sụp đổ 18.5×). Các thread thừa không giúp tính toán nhanh hơn mà tranh chấp gay gắt **băng thông bộ nhớ** và **cache L1/L2/L3** (dẫn đến cache thrashing). Đồng thời, chi phí chuyển ngữ cảnh (**context switching**) và đồng bộ hóa hàng đợi/barrier lock giữa các thread trong runtime tăng vọt, khiến CPU tốn chu kỳ chờ đợi thay vì xử lý dữ liệu.
