# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 20.7 | 100% |
| 2 | 20.7 | 100% |
| 4 | 20.6 | 100% |
| 8 | 20.6 | 100% |
| 16 | 20.7 | 100% |

**Best**: `-t 1` at 20.7 tok/s
**Slowest tested**: `-t 8` at 20.6 tok/s (1.00x spread)
**Against the physical-core default** (`-t 4`, 20.6 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Giải thích của tôi

Không có knee: đường cong phẳng (20.6–20.7 tok/s, chênh lệch 1.00×). Với `ngl=99` mọi
layer chạy trên MX130, CPU thread chỉ làm việc launch kernel GPU và copy vài byte mỗi
token; phần việc mà `-t` song song hoá không nằm trên critical path. Dòng "best = `-t 1`"
chỉ là nhiễu (0.1 tok/s). Để có đường cong thread có ý nghĩa, tôi chạy lại sweep với
`LAB_N_GPU_LAYERS=0` → `01-tuning-tg128.md`.
