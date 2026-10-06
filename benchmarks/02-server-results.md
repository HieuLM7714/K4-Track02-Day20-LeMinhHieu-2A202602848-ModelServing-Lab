# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 32 | 0.55 | 15000 | 21000 | 27000 | 8.5 | 0.0% |
| 50 | 28 | 0.49 | 31000 | 56000 | 56000 | 15.6 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.90x** (18% of linear) |
| P95 latency | **2.67x** |
| Effective concurrency at 50 users | 15.6 vs `--parallel 4` slots (occupancy/slot ratio 3.90) |

**Saturated.** Throughput delivered only 0.90x for 5x the offered load, and effective concurrency (15.6) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.90x while P95 moved 2.67x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Nhận định của tôi

**Server đã bão hoà ngay từ 10 users.** Effective concurrency ở 10 users là 8.5, gần gấp
đôi 4 slot, tức khoảng một nửa thời gian của mỗi request là chờ slot. Lên 50 users, offered
load tăng 5× nhưng throughput không tăng (0.55 → 0.49 RPS, 0.90×), trong khi P95 tăng 2.67×
(21 s → 56 s). Con số thuyết phục tôi nhất: trong lúc chạy 50 users, `requests_processing` =
4 và `requests_deferred` = 46 (`02-server-batching-u50.md`) — cả 50 users đều nằm trong hệ
thống, nhưng chỉ 4 request được tính toán.

Vì sao throughput còn *giảm*: lần chạy 50 users tình cờ có nhiều request `long-rag` hơn
(8/28 so với 4/32), cần prefill lâu hơn; và locust chỉ đếm những request hoàn thành trong
cửa sổ 60 s — ở 50 users nhiều request vẫn đang xếp hàng khi lần chạy kết thúc.

Phần latency tăng thêm là **queue time, không phải compute time**: các slot bận ~99%
(peak 3.96/4) nên thời gian tính toán mỗi request không thể tăng thêm; thứ tăng lên là
hàng đợi phía trước các slot (Little's Law: 15.6 request trong hệ thống so với 4 đang được
phục vụ → ~75% thời gian là chờ).

SLO tôi chọn cho laptop này: **P95 ≤ 25 s**. Ở 10 users P95 = 21 s → goodput ≈ toàn bộ
0.55 RPS. Ở 50 users ngay cả *median* đã là 31 s, nên chưa tới một nửa request đạt SLO
→ goodput < 0.25 RPS — chưa bằng một nửa so với 10 users.

Knob tôi đổi đầu tiên: **tăng tốc độ phục vụ**, không phải `--parallel`. Các slot đã đầy
và decode là bottleneck, thêm slot chỉ chia cùng một lượng decode throughput cho nhiều
request hơn (request nào cũng chậm hơn). `make tune` cho thấy CPU `-t 8` decode nhanh hơn
cấu hình GPU này 1.29×, nên đó là knob đầu tiên. Thứ hai là admission control (từ chối /
bỏ bớt request khi hàng đợi quá dài) để những request được nhận vẫn đạt SLO thay vì tất
cả cùng trễ.
