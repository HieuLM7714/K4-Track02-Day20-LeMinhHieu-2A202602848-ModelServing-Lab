# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 9406.5 | 9406.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 6531.4 | 6531.5 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 7544.1 | 7544.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **7827.3** · total **7827.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **goodput** is more useful than raw throughput because it focuses exclusively on the requests per second (RPS) that met the Target Time-to-Fill (TTFT) and Target Time-to-Poll (TPOT) targets.

The context explicitly states that throughput at saturation "ignores SLOs" (Service Level Objectives), meaning it does not consider how close the system is to its performance go

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** (specifically the wasted space consumed by non-contiguous pages) by storing the KV cache in non-contiguous pages. This design allows the engine to skip prefill entirely when a shared prefix is used, thereby optimizing memory usage and avoiding the fragmentation that would otherwise waste most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**.

This is because the context explicitly states that prefill is compute-bound and decode is memory-bandwidth-bound. By splitting them, the system can optimize the workload: the compute-bound part of prefill can be handled efficiently, while the memory-bound part of decode can be processed in parallel or


## Phần nào của N16-N19 là real

- N16 Cloud/IaC: **stub** — chạy trên laptop, không có hạ tầng cloud.
- N17 Data pipeline: **stub** — tài liệu là `TOY_DOCS` viết cứng trong `pipeline.py`.
- N18 Lakehouse: **stub** — không có lakehouse storage phía sau.
- N19 Vector + features: **stub** — không có embedding server, retrieval dùng fallback
  keyword overlap (vì vậy embed 0.0 ms, retrieve ≤ 0.1 ms).
- N20 Serving: **real** — `llama-server` (Qwen3.5 0.8B Q4_K_M, GPU MX130).

LLM chiếm 100% tổng thời gian (trung bình 7827 ms). Đúng như tôi dự đoán, vì retrieval là
stub nên LLM là stage thật duy nhất. Bên trong LLM, decode chiếm phần lớn: ví dụ query 1 =
prefill 151 tok / 872 ms so với decode 123 tok / 5994 ms. Để giảm latency 2×, tôi sẽ tấn
công **decode**: giới hạn độ dài câu trả lời (câu trả lời dài và đằng nào cũng bị cắt) và
dùng cấu hình CPU `-t 8` nhanh hơn (+26% decode). Kể cả vector search thật cũng chỉ thêm
vài mili giây.
