# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Lê Minh Hiếu
**MSSV:** 2A202602848
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

- **OS:** Windows 10 IoT Enterprise LTSC 2021 (AMD64)
- **CPU:** Intel Core i5-8265U @ 1.60 GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2 + FMA (không có AVX-512)
- **RAM:** 11.9 GB
- **Accelerator:** NVIDIA GeForce MX130 (2 GB, CUDA) — các lần chạy base dùng `ngl=99` (toàn bộ layer trên GPU)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-12.4-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (local, không dùng cloud).

**Setup story:** Lần đầu `pip install` bị treo, chạy lại thì được. Sau đó trình tải Hugging
Face đứng ở 0 MB (đo bằng curl chỉ ~0.5 MB/s), nên tôi chuyển từ Gemma 4 E2B (5.2 GB) sang
Qwen3.5 0.8B (0.9 GB), tải hai file GGUF bằng `curl` (`docs/MANUAL-DOWNLOAD.md`) rồi ghi
manifest bằng `download-model.py --skip-download`. Trên Windows các report sinh ra bị ghi
bằng cp1252, nên tôi chuyển sang UTF-8 và chạy `verify`/`metrics` với `PYTHONUTF8=1`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

Từ `benchmarks/01-quickstart-results.md` (`threads=4`, `ngl=99`, `max_tokens=64`, mỗi bản 10 request).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2992 | 722 / 833 | 49.0 / 49.0 | 3806 / 3919 / 3919 | 20.4 |
| UD-Q2_K_XL | 0.39 | 2990 | 867 / 917 | 60.5 / 60.5 | 4677 / 4725 / 4725 | 16.5 |

**Quan sát:** 2-bit nhỏ hơn 22% nhưng decode **chậm hơn 1.24×** và TTFT cũng chậm hơn →
không đáng dùng. Decode ở đây không bị chặn bởi bandwidth (~10 GB/s trên một GPU nhỏ), nên
chi phí dequantization của Q2 lớn hơn phần tiết kiệm được. Hỏi cùng câu cho cả hai: Q4 tính
đúng giờ tàu (12:15), Q2 trả lời "9:40" rồi lặp vô hạn (`quality-compare.txt`).

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

Từ `benchmarks/02-server-results.md` (`--parallel 4`, `ngl=99`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.55 | 15000 | 21000 | 27000 | 8.5 | 0.0% |
| 50 | 0.49 | 31000 | 56000 | 56000 | 15.6 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.90× (không tăng)
- **P95 tăng:** 2.67×
- **Effective concurrency ở 50 users:** 15.6 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`:** 3.96 / 4 slots (`requests_deferred` cao nhất 46; đo trong một lần chạy 50 users lặp lại vì `metrics` bị crash ở lần đầu — xem `02-server-batching-u50.md`)

**Saturation reading:** Server đã bão hoà từ 10 users: effective concurrency 8.5 > 4 slot.
Ở 50 users, 4 request đang xử lý trong khi 46 request bị deferred, slot bận ~99%, nên phần
P95 tăng thêm là queue time chứ không phải compute. Với SLO P95 ≤ 25 s, goodput giảm từ
0.55 xuống dưới 0.25 RPS. Knob tôi đổi đầu tiên là tăng tốc decode (CPU `-t 8`, 1.29× trong
`make tune`) chứ không phải thêm slot `--parallel`, vì thêm slot chỉ chia nhỏ cùng một
lượng decode throughput.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | chạy trên laptop local | stub |
| N17 Data pipeline | `TOY_DOCS` viết cứng trong `pipeline.py` | stub |
| N18 Lakehouse | không có | stub |
| N19 Vector + features | fallback keyword overlap, không có embedding server | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (trung bình 3 query, `benchmarks/03-integration-results.md`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 7827.3 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection:** LLM chiếm toàn bộ chi phí — đúng kỳ vọng vì retrieval là stub. Bên trong LLM,
decode chiếm phần lớn (query 1: prefill 872 ms so với decode 5994 ms). Để giảm latency 2×,
tôi sẽ giới hạn độ dài câu trả lời và dùng decode CPU `-t 8` nhanh hơn; vector search thật
cũng chỉ thêm vài mili giây.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

**Change:** tắt GPU offload, chạy decode trên CPU với số thread đã tune:
`ngl=99` (MX130) → `LAB_N_GPU_LAYERS=0`, `-t 8`.

```
before:  20.7 tok/s  (tg128, ngl=99, GPU MX130)      benchmarks/01-tuning-tg128-gpu.md
after:   26.8 tok/s  (tg128, ngl=0,  CPU -t 8)       benchmarks/01-tuning-tg128.md
speedup: 1.29x decode
```

Bench end-to-end, Q4_K_M (`01-quickstart-results.md` so với `01-quickstart-cpu-t8.md`):
TPOT P50 49.0 → 38.8 ms, nhưng TTFT P50 lại **tệ hơn**, 722 → 923 ms; E2E P50 3806 → 3325 ms (1.14×).

**Tại sao nó work:**

Kỳ vọng từ deck là "GPU nhanh hơn". Trên laptop này điều đó sai với decode, và lý do nằm ở
thứ đang chặn decode. Ở 20.7 tok/s, GPU đọc 0.5 GB × 20.7 ≈ 10 GB/s weight — thấp hơn nhiều
so với memory bandwidth của MX130 — nên GPU **không** bị chặn bởi bandwidth. Nó bị chặn vì là
một GPU entry-level rất nhỏ chạy model 0.8B từng token một: mỗi token là một chuỗi dài các
kernel nhỏ (nhân ma trận-vector), quá nhỏ để lấp đầy GPU. Overhead launch/đồng bộ mỗi kernel
cộng với compute yếu của GPU tạo thành trần tốc độ — đó là lý do thread sweep trên GPU phẳng
hoàn toàn. Trên CPU, cùng phép nhân ma trận-vector đó chạy bằng AVX2 trực tiếp từ RAM và
cache, không có overhead launch. Đường cong thread (16.0 → 24.3 → 26.1 → 26.8 tok/s với
1/2/4/8 thread, rơi xuống 20.0 ở 16 thread) cho thấy khi 2–4 core đã làm memory bus bận thì
thêm core lợi rất ít; vượt quá 8 CPU logic thì oversubscription khiến mỗi barrier của ggml
phải chờ thread bị OS tạm dừng.

Prefill đi theo chiều ngược lại, và kết quả TTFT xác nhận cơ chế này. Prefill xử lý toàn bộ
token của prompt cùng lúc dưới dạng nhân ma trận-ma trận — compute-bound và đủ lớn để GPU bận
— nên GPU vẫn thắng TTFT (722 so với 923 ms). Với prompt ngắn và câu trả lời 64 token của lab
này, decode chiếm phần lớn thời gian nên CPU thắng end-to-end; với prompt RAG dài, nhiều khả
năng GPU sẽ thắng lại. Lưu ý: phần serving và load test (§3) chạy với cấu hình GPU mặc định,
trước khi tôi có kết quả này.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

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

Claude Code (Anthropic): chạy các lệnh của lab, debug lỗi pip / tải Hugging Face bị treo và lỗi
encoding trên Windows, soạn nháp các phần nhận xét và §2–§5 dựa trên số liệu do chính máy tôi
sinh ra. Mọi số liệu đều từ script chạy trên laptop này; không sửa tay con số nào.
