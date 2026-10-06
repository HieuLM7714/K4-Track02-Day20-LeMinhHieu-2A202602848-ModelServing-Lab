# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3495 | 923 / 988 | 38.8 / 42.2 | 3325 / 3595 / 3595 | 25.8 |
| UD-Q2_K_XL | 0.39 | 3607 | 1039 / 1107 | 39.3 / 40.0 | 3544 / 3630 / 3630 | 25.4 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `Q4_K_M` decode within 2% of each other here, for 0.11 GB difference on disk.

## Nhận xét của tôi

Đây là lần chạy bổ sung, không phải baseline chính: cùng bench nhưng với
`LAB_N_GPU_LAYERS=0 LAB_N_THREADS=8` (điểm tốt nhất từ `make tune`). So với baseline GPU
(`01-quickstart-results.md`) cho Q4_K_M: decode 20.4 → 25.8 tok/s (TPOT P50 49.0 → 38.8 ms),
nhưng TTFT P50 lại *tệ hơn*, 722 → 923 ms; E2E P50 cải thiện 3806 → 3325 ms (1.14×).
Decode thắng trên CPU, prefill vẫn thắng trên GPU — xem REFLECTION §5. Trên CPU hai bản
quant decode chênh nhau chưa tới 2%, nên 2-bit vẫn không mang lại gì.
