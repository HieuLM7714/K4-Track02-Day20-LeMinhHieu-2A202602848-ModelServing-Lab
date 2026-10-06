# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 16.0 | 60% |
| 2 | 24.3 | 91% |
| 4 | 26.1 | 97% |
| 8 | 26.8 | 100% |
| 16 | 20.0 | 74% |

**Best**: `-t 8` at 26.8 tok/s
**Slowest tested**: `-t 1` at 16.0 tok/s (1.67x spread)
**Against the physical-core default** (`-t 4`, 26.1 tok/s): 1.03x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Giải thích của tôi

Sweep này chạy **chỉ trên CPU** (`LAB_N_GPU_LAYERS=0`). Sweep mặc định đưa toàn bộ layer
lên MX130 và cho kết quả phẳng (20.6–20.7 tok/s ở mọi `-t`, xem `01-tuning-tg128-gpu.md`),
nên không nói được gì về số thread. Trên CPU đường cong có hình dạng rõ ràng:

- `-t 1 → 2`: 16.0 → 24.3 tok/s (**1.52×**). Một core không phát lệnh load đủ nhanh để
  giữ bộ nhớ bận; thêm core thứ hai gần như gấp đôi lượng byte được đọc cùng lúc.
- `-t 2 → 4`: +7%, `-t 4 → 8`: +3%. **Knee nằm ở 2–4 thread.** Decode phải đọc toàn bộ
  0.5 GB weight cho mỗi token, nên khi vài core đã làm DRAM bận, thêm core chỉ là thêm
  người chờ trên cùng một memory bus. 8 thread nhỉnh hơn 4 một chút vì hyper-thread trên
  cùng core che được một phần memory latency, nhưng chúng dùng chung execution unit và
  cache nên lợi ích nhỏ.
- `-t 16`: giảm còn 20.0 tok/s (74%). 16 thread trên 8 CPU logic khiến OS phải chia thời
  gian; ggml đồng bộ tất cả thread ở một barrier sau mỗi phép tính, nên mỗi bước phải chờ
  thread vừa bị OS tạm dừng. Oversubscription biến thành thời gian chờ.

Tốt nhất: `-t 8` với 26.8 tok/s. Kết quả này còn nhanh hơn cả lần chạy GPU (20.7 tok/s)
— xem REFLECTION §5.
