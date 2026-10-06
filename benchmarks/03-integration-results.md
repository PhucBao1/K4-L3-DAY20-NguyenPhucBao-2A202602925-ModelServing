# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 4649.5 | 4649.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 3768.9 | 3769.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 5346.1 | 5346.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **4588.2** · total **4588.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Phần nào là thật (real), phần nào là giả lập (stub)

| Ngày | Thành phần | Real hay stub |
|---|---|---|
| N16 Cloud/IaC | hạ tầng | **stub** — mọi thứ chạy trên laptop, không deploy IaC |
| N17 Data pipeline | nạp dữ liệu | **stub** — tài liệu là danh sách viết sẵn trong `pipeline.py` |
| N18 Lakehouse | lưu trữ | **stub** — không có lakehouse, tài liệu nằm trong bộ nhớ |
| N19 Vector + features | embed + tìm kiếm | **stub** — giữ nguyên `STUB 1/2` trong `pipeline.py`: không có embedding, tìm bằng trùng từ khoá |
| N20 Serving | `llama-server` | **real** — Gemma 4 E2B UD-Q4_K_XL, CPU, `--parallel 4` |

**Bước tốn thời gian nhất có như tôi đoán không?** Có — LLM chiếm 100% của 4,588 ms trung bình,
vì embed/retrieve là stub (0 ms). Điều đáng chú ý hơn: **ngay trong bước LLM, phần đọc prompt
(prefill) đã chiếm 40–55%**, dù context lấy về rất ngắn (113–149 token). Ví dụ query 3: prefill
2,893 ms, sinh câu trả lời 2,434 ms. CPU này chỉ đọc prompt được ~40–75 token/giây, nên mỗi đoạn
tài liệu lấy thêm đều làm người dùng chờ lâu hơn rõ rệt.

**Muốn giảm một nửa độ trễ**, tôi sẽ tấn công prefill trước:
1. Đặt system prompt cố định ở đầu để llama.cpp dùng lại phần đã tính (prefix cache), khỏi tính lại mỗi lần.
2. Lấy ít đoạn tài liệu hơn, ngắn hơn.
3. Dùng `-t 8` cho tải nặng prefill (sweep của tôi cho thấy prefill nhanh nhất ở 8 thread).

Sau đó mới giới hạn độ dài câu trả lời (~94 ms mỗi token). Một bước embedding thật ở N19 chỉ tốn
vài chục ms — không đáng kể so với 4.6 s của LLM trên máy này.
