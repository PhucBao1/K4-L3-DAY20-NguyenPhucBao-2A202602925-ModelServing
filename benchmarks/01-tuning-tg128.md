# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **10 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 8.9 | 78% |
| 5 | 11.4 | 100% |
| 10 | 10.9 | 95% |
| 12 | 10.6 | 93% |
| 24 | 8.7 | 76% |

**Best**: `-t 5` at 11.4 tok/s
**Slowest tested**: `-t 24` at 8.7 tok/s (1.31x spread)
**Against the physical-core default** (`-t 10`, 10.9 tok/s): 1.05x

Use this in your run:

```bash
LAB_N_THREADS=5 make bench
```

## Đo thêm: decode và prefill theo số thread (cùng binary, `llama-bench -r 2`)

Tôi chạy thêm ngay sau `make tune` để so sánh hai loại công việc khác nhau:
- **tg128 (decode)** = sinh từng token trả lời.
- **pp512 (prefill)** = đọc một prompt 512 token.

| threads | tg128 (tok/s) | pp512 (tok/s) |
|--:|--:|--:|
| 1 | 8.51 | 20.40 |
| 2 | 10.76 | 29.39 |
| 4 | 11.29 | 36.12 |
| 5 | 11.35 | 36.86 |
| 6 | 11.15 | 38.65 |
| 8 | 10.67 | **43.40** |
| 10 | 10.11 | 39.24 |
| 12 | 8.78 | 41.17 |

(Ở 12 thread, số decode dao động giữa các lần chạy: 10.6 trong `make tune`, 8.8 ở đây. Đây là
chip laptop 15 W nên dễ bị giảm xung khi nóng → sai số khoảng ±1 tok/s ở nhiều thread.)

## Giải thích của tôi

**Điểm tốt nhất của decode là 4–5 thread, không phải 10 core.** Chỉ 1 thread đã đạt 78% tốc độ
tốt nhất (8.9 so với 11.4 tok/s), và tăng từ 5 lên 10 thread còn làm **chậm đi**.

**Vì sao?** Mỗi lần sinh 1 token, CPU phải đọc gần như toàn bộ ~3 GB trọng số model từ RAM, nhưng
với mỗi byte đọc vào chỉ làm rất ít phép tính. Nên nút thắt là **tốc độ đọc RAM (memory
bandwidth)**, không phải số core. Giống như 1 cái vòi nước: vài người hứng là vòi đã chảy hết
cỡ, thêm người hứng nữa cũng không ra thêm nước — chỉ thêm chen lấn. Cụ thể, thêm thread chỉ làm
tăng chi phí đồng bộ (các thread phải chờ nhau sau mỗi bước tính) và tranh nhau bộ nhớ đệm L3.

**Riêng CPU này còn có lý do thứ hai:** i5-1235U có 2 core mạnh (P-core) và 8 core yếu (E-core).
llama.cpp chia việc **đều** cho mọi thread rồi chờ thread chậm nhất xong. Khi có thread chạy trên
E-core chậm, các P-core nhanh phải ngồi chờ. Ở 12–24 thread thì còn tranh core với hệ điều hành.

**Bằng chứng đây đúng là do băng thông RAM:** cùng model, cùng binary, nhưng **prefill vẫn nhanh
lên tới 8 thread** (20 → 43 tok/s, gấp 2.1 lần). Prefill xử lý cả đoạn prompt một lúc, mỗi byte
trọng số đọc vào được dùng cho nhiều token → bị giới hạn bởi sức tính, nên thêm core (kể cả
E-core) vẫn có ích. Hai loại công việc, hai đường cong ngược nhau.

**Kết luận:** decode dùng `-t 5` (11.4 so với 10.9 tok/s ở mặc định 10 thread, nhanh hơn 1.05
lần). Mức tăng nhỏ vì nút thắt là RAM chứ không phải thread. Muốn decode nhanh hẳn thì phải
**đọc ít byte hơn mỗi token** (model nén nhỏ hơn) hoặc **một lần đọc dùng cho nhiều người**
(continuous batching).
