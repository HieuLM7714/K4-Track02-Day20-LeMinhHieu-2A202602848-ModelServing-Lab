# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 9404 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Nhận xét của tôi

Batch width cao nhất là **3.96/4 slot** (99%) — continuous batching đang gom 4 request vào
mỗi bước decode gần như suốt cửa sổ đo.

Ghi chú: trong lần `load-50` của tôi, `make metrics` bị crash ngay khi khởi động (chạy trong
background job trên Windows → console dùng encoding cp1252). Tôi chạy lại `metrics` với
`PYTHONUTF8=1` song song với một lần locust 50 users thứ hai giống hệt, CSV của lần đó ghi
ra ngoài `benchmarks/`, để không ghi đè kết quả `locust-50` khớp với screenshot của tôi.

Con số này không khớp với effective concurrency 15.6 trong `02-server-results.md`, và
cũng không nên khớp: hai số đếm hai thứ khác nhau. Busy slots = số request đang được
*tính toán*; concurrency theo Little's Law = số request *trong hệ thống*, kể cả hàng đợi.
Phần chênh (15.6 − 4 ≈ 12) chính là hàng đợi. Về kích thước hàng đợi tôi tin gauge của
server hơn: `requests_deferred` lên tới 46, tức gần như cả 50 users đang chờ hoặc đang chạy.
Con số 15.6 của locust bị thấp vì nó chỉ tính 28 request *đã hoàn thành* trong 60 s —
những request chậm nhất còn đang chạy bị bỏ ra.
