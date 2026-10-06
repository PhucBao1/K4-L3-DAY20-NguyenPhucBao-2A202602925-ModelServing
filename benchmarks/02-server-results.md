# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=10` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 18 | 0.39 | 23000 | 36000 | 36000 | 7.9 | 0.0% |
| 50 | 20 | 0.36 | 19000 | 55000 | 55000 | 7.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.93x** (19% of linear) |
| P95 latency | **1.53x** |
| Effective concurrency at 50 users | 7.9 vs `--parallel 4` slots (occupancy/slot ratio 1.98) |

**Saturated.** Throughput delivered only 0.93x for 5x the offered load, and effective concurrency (7.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.93x while P95 moved 1.53x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 18 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Nhận định của tôi

**Server đã quá tải ngay từ 10 users.** Bằng chứng rõ nhất: tăng tải lên 5 lần (10 → 50 users)
mà số request xử lý được mỗi giây **không tăng chút nào** (0.39 → 0.36 RPS, tức 0.93×). Server
đã chạy hết sức ở 10 users; người dùng thêm vào chỉ làm hàng đợi dài ra.

**Tách thời gian chờ và thời gian tính (Little's Law):** số request nằm trong hệ thống =
RPS × độ trễ trung bình = **7.9** ở cả 10 và 50 users, trong khi server chỉ có **4 slot**. Vậy
trung bình ~4 request đang được tính và ~4 request đang **xếp hàng**. Một câu trả lời 64 token
lúc server rảnh chỉ mất ~5.5 s (`make bench`, E2E P50 = 5,485 ms), nhưng dưới tải trung vị là
19–23 s → phần lớn thời gian người dùng chờ là **xếp hàng**, không phải tính toán.

**P95 tăng 1.53× (36 s → 55 s) trong khi throughput đứng yên** — đây chính là lập luận goodput:
sau điểm bão hoà, thêm tải không mua thêm được throughput, chỉ làm độ trễ tệ hơn. Nếu đặt SLO là
"P95 ≤ 30 s" thì ở 10 users đã trượt (36 s), ở 50 users còn trượt xa hơn → goodput@SLO gần như
về 0 dù RPS vẫn ~0.36–0.39.

**Lưu ý khi đọc số:** mỗi lần test chỉ 60 s, nên request nào cần hơn ~60 s thì chưa kịp xong và
**không được tính**. P95 55 s ở 50 users đang chạm trần 60 s → độ trễ thật còn tệ hơn. Số mẫu
cũng ít (18 và 20 request), nên các percentile chỉ mang tính tham khảo.

**Batching có giúp không?** Có. Trong lần đo `make metrics` lúc chạy load-50 (lần chạy đầu
tiên, xem `02-server-batching-u50.md`), 4 slot luôn bận (3.85/4), 46 request đang chờ, và server
sinh ~22 tok/s tổng so với ~11 tok/s khi phục vụ 1 người → batching **gấp đôi** tổng throughput.
Nhưng mỗi người thì chậm đi (mỗi token ~200 ms thay vì ~90 ms), vì CPU bị giới hạn bởi băng thông
RAM: đọc trọng số một lần cho 4 người, nhưng mỗi bước dài hơn.

**Knob đầu tiên tôi sẽ đổi để tăng goodput@SLO:** **giới hạn hàng đợi** (từ chối hoặc báo bận khi
đã có ~4–8 request), chứ **không** tăng `--parallel`. Thêm slot chỉ tăng tổng tok/s một chút
nhưng làm mỗi request chậm hơn → ít request đạt hạn hơn. Knob thứ hai: giảm số byte phải đọc
cho mỗi token (giới hạn `max_tokens`, hoặc dùng model nhỏ hơn cho câu trả lời ngắn).
