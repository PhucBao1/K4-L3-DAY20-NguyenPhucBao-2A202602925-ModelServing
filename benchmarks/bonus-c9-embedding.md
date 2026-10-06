# Bonus C9 (B5) - Phục vụ embedding (embedding serving)

Máy: i5-1235U, chỉ CPU · llama.cpp `b10488` · model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` chạy ở chế độ
pooling (`make serve-embed` = `--embedding --pooling mean`, `-t 10`, vector 1536 chiều).

> **Giới hạn:** đây là model chat được dùng tạm làm bộ tạo embedding (lấy trung bình trạng thái
> ẩn), **không phải** model embedding chuyên dụng (Qwen3-Embedding, BGE-M3, EmbeddingGemma). Nên
> chất lượng tìm kiếm ở đây không có nhiều ý nghĩa; thứ được đo là **chế độ làm việc**: chỉ có
> prefill (đọc văn bản một lượt), không có KV cache dùng lại, không có vòng lặp sinh token.

## Kiểm tra tìm kiếm (`make embed-demo`)

Câu hỏi: *"Does embedding serving use a KV cache and a decode loop like chat serving?"*

| hạng | cosine | tài liệu |
|--:|--:|---|
| 1 | 0.843 | Embedding serving is prefill-bound: one forward pass, no KV cache, no decode loop. |
| 2 | 0.780 | RadixAttention reuses a shared prompt prefix ... |
| 3 | 0.747 | Speculative decoding drafts several tokens ... |

Hạng 1 đúng, nhưng khoảng cách với tài liệu không liên quan rất nhỏ (0.84 so với 0.75–0.78). Đây
là đặc điểm của model chat dùng làm embedding: mọi vector đều "chụm" vào một hướng.

## Throughput theo kích thước batch

Mỗi văn bản dài 10–20 token (đếm bằng `/tokenize`). Mỗi điểm lấy trung vị của 3 lần gửi (tôi
chạy lại sweep của demo; bản gốc `make embed-demo` cho 3.2 / 3.6 / 3.6 / 1.9 / 1.9 texts/s).

| batch | tokens | `--parallel 4` (mặc định) ms | texts/s | tok/s | `--parallel 16` ms | texts/s | tok/s |
|--:|--:|--:|--:|--:|--:|--:|--:|
| 1 | 15 | 293 | 3.41 | 51.2 | 290 | 3.45 | 51.8 |
| 2 | 29 | 567 | 3.53 | 51.2 | 639 | 3.13 | 45.4 |
| 4 | 59 | 1,492 | 2.68 | 39.6 | 2,210 | 1.81 | 26.7 |
| 8 | 120 | 4,281 | 1.87 | 28.0 | 4,397 | 1.82 | 27.3 |
| 16 | 240 | 8,423 | 1.90 | 28.5 | 8,558 | 1.87 | 28.0 |

## Phát hiện

**Trên CPU này, gom nhiều văn bản vào một batch KHÔNG làm nhanh hơn — còn chậm đi.** Thời gian
tăng tỉ lệ thuận với tổng số token (~35 ms/token khi batch ≥ 8), nên số văn bản/giây đứng yên
hoặc giảm ~45% từ batch 1–2 lên batch 8–16. Điều này **ngược** với slide: "throughput đến từ
batch tĩnh lớn".

**Giả thuyết đầu tiên của tôi — thiếu slot — đã sai.** Server embedding mặc định chỉ có 4 slot,
nên tôi nghĩ batch 8/16 phải xếp hàng chia đợt. Chạy lại với `--parallel 16` thì kết quả y hệt
(1.82–1.87 texts/s ở batch 8/16). Vậy nút thắt không nằm ở bộ lập lịch.

**Cơ chế thật:** tạo embedding chỉ là prefill — xử lý cả đoạn văn bản một lúc, tức là phép nhân
ma trận × ma trận, cần **sức tính**. Ngay cả một văn bản 15 token cũng đã đủ làm bận cả 10
thread (`make tune` cho thấy prefill trên CPU này bị giới hạn bởi sức tính: ~39–43 tok/s ở 8–12
thread). CPU đã chạy hết công suất từ batch 1, nên không còn chỗ trống nào để batching lấp vào.
Thêm văn bản chỉ là thêm việc tương ứng, cộng thêm chi phí riêng cho từng văn bản (mask
attention riêng, pooling riêng, dữ liệu tạm tràn khỏi bộ nhớ đệm).

**So với chat (track 02):** continuous batching làm throughput chat **tăng gấp đôi** (~22 tok/s
tổng khi 3.85 slot bận, so với ~11 tok/s khi phục vụ 1 người). Vì decode là ma trận × vector,
bị giới hạn bởi băng thông RAM — đọc trọng số một lần có thể phục vụ 4 người gần như miễn phí.
Embedding thì không có gì để "chia sẻ" thêm: trọng số đã được dùng lại cho mọi token trong cùng
một văn bản rồi.

**Vậy quy tắc "chat và embedding cần chiến lược batching ngược nhau" phụ thuộc phần cứng.** Trên
GPU, một lượt 15 token để trống phần lớn nhân tính toán, nên batch lớn giúp lấp đầy — đó là lý
do slide khuyên như vậy. Trên CPU đã chạy hết công suất ngay ở batch 1, cách đúng là batch nhỏ
(1–2) cho độ trễ thấp, muốn tăng throughput thì thêm core/thêm máy. Khi đặt cả hai sau một
autoscaler: máy chat nên co giãn theo độ dài hàng đợi / số slot bận, máy embedding nên co giãn
theo % CPU. Nếu để chung một máy, một đợt embedding sẽ cướp đúng phần sức tính mà prefill của
chat (TTFT) đang cần.
