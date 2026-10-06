# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** THIỀU QUANG VINH
**MSSV:** 2A202602877
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (WSL2 Ubuntu 24.04 LTS / Linux 6.18)
- **CPU:** 12th Gen Intel(R) Core(TM) i7-12650H
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 11.5 GB
- **Accelerator:** NVIDIA GeForce RTX 3070 Laptop GPU, 8192 MiB (baseline chạy CPU-only, ngl: 0)
- **llama.cpp asset đã tải:** llama-b10488-bin-ubuntu-vulkan-x64.tar.gz
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** Laptop cá nhân (WSL2 Ubuntu)
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): Khi chạy `make bench`, `llama-server` bị crash với mã lỗi 127 do thiếu thư viện OpenMP: `error while loading shared libraries: libgomp.so.1: cannot open shared object file`. Đã khắc phục bằng cách cài gói qua `sudo apt install -y libgomp1`. Sau đó server khởi động và benchmark bình thường.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2138 | 178 / 188 | 20.9 / 21.2 | 1496 / 1518 / 1518 | 47.8 |
| UD-Q2_K_XL | 0.39 | 2084 | 219 / 249 | 20.3 / 20.6 | 1495 / 1518 / 1518 | 49.3 |

**Quan sát** (≤ 60 chữ): 2-bit decode nhanh hơn 1.03x (49.3 vs 47.8 tok/s) và nhẹ hơn 0.11 GB, nhưng prefill (TTFT) chậm hơn ~1.3x (219 vs 178 ms). Khi hỏi cùng câu hỏi ở lượt 2, bản 2-bit trả lời cụt lủn (26 tokens) và bị hallucinated so với câu trả lời đầy đủ của bản 4-bit (86 tokens). Không đáng đánh đổi.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.06 | 8300 | 12000 | 13000 | 8.6 | 0.0% |
| 50 | 1.28 | 28000 | 41000 | 42000 | 33.3 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.20×
- **P95 tăng:** 3.42×
- **Effective concurrency ở 50 users:** 33.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.93 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa ở mức ≤ 50 users: tải tăng 5x nhưng RPS chỉ tăng 1.20x, trong khi P95 bùng nổ 3.42x (12s -> 41s). Độ trễ tăng thêm là queue time vì effective concurrency đạt 33.3 (gấp 8.32x số 4 slot) và deferred=46. Để nâng goodput@SLO (P95 ≤ 15s), đổi -ngl bật GPU offload trước để tăng tốc giải phóng slot.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Cloud infrastructure / IaC | stub |
| N17 Data pipeline | Data ingestion / ETL | stub |
| N18 Lakehouse | Storage / Lakehouse table | stub |
| N19 Vector + features | Vector search / Embeddings | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 3059.8 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck hoàn toàn ở LLM (3059.8 ms) do embed/retrieve dùng keyword overlap in-memory (0.0 ms), đúng như kỳ vọng. Để giảm latency pipeline 2x, cần tấn công vào LLM bằng cách bật GPU offload (-ngl 99) để tăng tốc decode và siết max_tokens ngắn gọn hơn.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Hạ số thread từ logical cores (-t 16) về physical cores (-t 8) khi decode CPU (tg128).

```
before:  21.2 tok/s
after:   49.9 tok/s
speedup: 2.35×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Decode (tg128) là memory-bandwidth bound vì mỗi token sinh ra phải nạp lại toàn bộ trọng số mô hình từ RAM vào thanh ghi. Ở 8 physical cores, mỗi core tận dụng tối đa cache L1/L2 và bus bộ nhớ mà không bị tranh chấp, đạt tốc độ đỉnh 49.9 tok/s.

Khi nâng lên 16 threads (SMT/hyper-threading), 2 thread ảo trên cùng core tranh chấp kênh bộ nhớ và gây cache thrashing. Đồng thời chi phí chuyển ngữ cảnh (context switching) và đồng bộ hóa barrier giữa các thread tăng vọt, làm hiệu năng sụt giảm nghiêm trọng xuống còn 21.2 tok/s.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _(để trống nếu bạn không làm phần này)_

**Numbers:**

```
before:  (để trống nếu không làm)
after:   (để trống nếu không làm)
speedup: 1.00×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Bản 2-bit tuy tiết kiệm được 0.11 GB dung lượng và decode tương đương, nhưng prefill trên CPU lại chậm hơn do chi phí dequantization quá lớn, và chất lượng sinh câu trả lời ở lượt 2 bị hallucinated và cụt lủn.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng Antigravity IDE (Gemini) để hỗ trợ phân tích kết quả benchmark, đọc logs và định dạng các bảng số liệu trong báo cáo. Toàn bộ quá trình chạy benchmark, load test và inference phục vụ đều được thực thi trực tiếp trên máy local.
