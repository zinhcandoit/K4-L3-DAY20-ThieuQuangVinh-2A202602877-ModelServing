# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2138 | 178 / 188 | 20.9 / 21.2 | 1496 / 1518 / 1518 | 47.8 |
| UD-Q2_K_XL | 0.39 | 2084 | 219 / 249 | 20.3 / 20.6 | 1495 / 1518 / 1518 | 49.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.03x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Your observation

- **Tốc độ & Cơ chế:** Máy chạy thuần CPU (`ngl: 0`), hệ thống rơi vào trạng thái **compute-bound** do gánh nặng dequantize của định dạng 2-bit. Bản 4-bit có tốc độ prefill nhanh hơn ~1.5× ở lượt 1 (223.5 vs 149.3 tok/s); tốc độ decode giữa 2 bản tương đương nhau (~42–48 tok/s). Cả hai đều tận dụng tốt prefix caching qua LCP similarity (~90%).
- **Chất lượng sinh:** Ở lượt 2 ("How's the weather today?"), bản 4-bit trả lời đầy đủ, mạch lạc, trung thực (86 tokens), trong khi bản 2-bit suy giảm chất lượng rõ rệt, câu trả lời bị hallucinated. (chỉ 26 tokens).
- **Kết luận:** **Không đáng đánh đổi**. Bản 2-bit chỉ tiết kiệm 0.11 GB RAM nhưng prefill chậm hơn và câu trả lời kém hữu ích; bản 4-bit (`Q4_K_M`) tối ưu hơn toàn diện.

