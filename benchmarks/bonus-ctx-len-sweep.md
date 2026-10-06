# Bonus - Context-length sweep (prefill cost)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=8` `ngl=0` · RAM 11.5 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 236.0 | 1084.9 | 1.00x |
| 1024 | 201.7 | 5076.6 | 1.17x |
| 2048 | 188.8 | 10847.5 | 1.25x |
| 4096 | 174.6 | 23466.1 | 1.35x |

At 4096 tokens, prefill costs **23466 ms** --
1.35x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

- **Điểm uốn và sự bùng nổ phi tuyến:** Ở mức prompt ngắn (256 tokens), tốc độ prefill đạt 236.0 tok/s (~1.08s). Tuy nhiên khi mở rộng prompt lên 4096 tokens, throughput prefill giảm xuống 174.6 tok/s và thời gian prefill bùng nổ lên **23,466 ms (~23.5s)**, gấp **1.35× so với mức tăng tuyến tính** (vượt xa mức dự đoán tuyến tính ~17.3s). Đây là bằng chứng rõ ràng cho thấy độ phức tạp $O(N^2)$ của ma trận self-attention và chi phí cấp phát KV cache bắt đầu bộc lộ rõ rệt và chiếm ưu thế so với các lớp chiếu $O(N)$.
- **Tác động lên RAG Pipeline:** Toàn bộ thời gian prefill này được tính trực tiếp vào TTFT (Time To First Token) trước khi người dùng nhìn thấy bất kỳ token nào được sinh ra. Với độ trễ TTFT lên tới 23.5 giây ở 4096 tokens, prefill hoàn toàn áp đảo thời gian decode (chỉ mất ~1–2s). Vì vậy, hệ thống RAG không thể tùy tiện nhồi nhét context vào prompt: trên CPU chỉ nên duy trì context dưới 1024 tokens (khoảng 2–3 chunks ngắn) để giữ TTFT trong ngưỡng chấp nhận được (<5s).
