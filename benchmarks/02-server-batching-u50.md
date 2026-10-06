# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 28 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.85 of 4 slots (96%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1972 |

Highest sampled value was **3.85 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Nhận xét của tôi

Số slot bận cao nhất là **3.85 / 4**, `requests_processing = 4` và `requests_deferred` ở mức 44–46
gần như suốt 1 phút. Đây là bằng chứng **continuous batching đang hoạt động**: mỗi bước decode
xử lý cùng lúc ~4 người. (Chỉ số này là trung bình cộng dồn từ lúc bật server, nên nó tăng dần
3.70 → 3.85 chứ không nhảy — lần test 10 users trước đó cũng đã giữ các slot gần đầy.)

Con số này **khác** với effective concurrency 7.9–12.8 trong `02-server-results.md` (tuỳ lần chạy), và khác là
**đúng**: Little's Law đếm **mọi** request đang ở trong hệ thống (4 đang được xử lý + những cái
đang xếp hàng), còn `n_busy_slots_per_decode` chỉ đếm request **đang được tính**. Ghép hai số lại
thấy rõ: ~4 đang được phục vụ, phần còn lại đang chờ.

- Muốn biết "batching có chạy không" → tin chỉ số của server (đo ngay trong bộ lập lịch).
- Muốn biết "xếp hàng bao nhiêu" → tin Little's Law — và con số đó còn **thấp hơn thực tế**, vì 46
  request chưa xong lúc kết thúc không được locust tính vào.
