# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2992 | 722 / 833 | 49.0 / 49.0 | 3806 / 3919 / 3919 | 20.4 |
| UD-Q2_K_XL | 0.39 | 2990 | 867 / 917 | 60.5 / 60.5 | 4677 / 4725 / 4725 | 16.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.24x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Nhận xét của tôi

**Trên máy này 2-bit không đáng dùng.** `UD-Q2_K_XL` nhỏ hơn 22% (0.39 so với 0.50 GB)
nhưng lại *chậm hơn* ở mọi chỉ số: decode 16.5 so với 20.4 tok/s (TPOT P50 60.5 so với
49.0 ms), TTFT P50 867 so với 722 ms, E2E P50 4677 so với 3806 ms.

Lần chạy này đưa toàn bộ layer lên MX130 (`ngl=99`). Đọc 0.50 GB weight cho mỗi token ở
20.4 tok/s chỉ tương đương ~10 GB/s, thấp hơn nhiều so với memory bandwidth danh định của
card, nên decode ở đây **không** bị chặn bởi bandwidth mà bị chặn bởi compute / overhead
của từng kernel trên một GPU entry-level nhỏ. Trong trường hợp đó, phần dequantization
phức tạp hơn của Q2_K tốn nhiều hơn phần byte nó tiết kiệm được. Khi chạy trên CPU
(`ngl=0 -t 8`, xem `01-quickstart-cpu-t8.md`) hai bản quant decode chênh nhau chưa tới 2%
(25.8 so với 25.4 tok/s) — 2-bit vẫn không thắng.

Chất lượng — hỏi cùng câu cho cả hai server, temperature 0 (`quality-compare.txt`):
"Tàu chạy lúc 9:40, đi mất 2 giờ 35 phút — mấy giờ tới?" → Q4 trả lời **12:15** (đúng).
Q2 trả lời "9:40" rồi lặp `9:40 + 2:35 = 9:40 + 2:35 ...` tới hết token. Với câu hỏi giải
thích, Q2 cũng lặp lại chính câu của nó. Nhỏ hơn + chậm hơn + trả lời tệ hơn rõ rệt → giữ
Q4_K_M; tiết kiệm 0.11 GB RAM không có ý nghĩa trên máy 12 GB.
