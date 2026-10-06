# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 2041 | 338 / 571 | 90.3 / 94.9 | 5485 / 6549 / 6549 | 11.1 |
| UD-Q2_K_XL | 2.24 | 2023 | 741 / 882 | 81.2 / 82.5 | 5855 / 6032 / 6032 | 12.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.11x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Nhận xét của tôi

**Tốc độ:** Bản 2-bit (UD-Q2_K_XL) sinh token nhanh hơn **1.11 lần** (11.1 → 12.3 tok/s, mỗi token
từ 90.3 → 81.2 ms) và nhẹ hơn 0.73 GB (~25%).

- Nhanh hơn ít hơn mức nhỏ đi (file nhỏ 1.33 lần mà chỉ nhanh 1.11 lần). Lý do: không phải mọi
  phần của model đều bị nén xuống 2-bit (bảng embedding lớn của Gemma và một số lớp quan trọng
  vẫn giữ độ chính xác cao), và CPU phải tốn thêm công "giải nén" 2-bit → số thực trước khi tính.
- **TTFT lại chậm hơn hẳn** (P50 338 → 741 ms, gấp 2.2 lần). TTFT là giai đoạn đọc prompt
  (prefill) — giai đoạn này bị giới hạn bởi sức tính của CPU, và phép tính trên dữ liệu 2-bit
  tốn công hơn 4-bit. Lần chạy `make bench` trước đó cũng cho kết quả giống vậy (441 → 769 ms),
  nên đây không phải ngẫu nhiên.

**Chất lượng** (hỏi cùng 3 câu cho cả hai bản, temperature 0): cả hai đều tính đúng giờ tàu đến
(17:25) và giải thích KV cache bằng tiếng Việt ổn. Nhưng bản 2-bit trả `"year": "1889"` dạng
**chữ** thay vì **số** — lỗi nhỏ nhưng sẽ làm hỏng chương trình nào đọc JSON theo đúng kiểu.

**Có đáng dùng 2-bit không?** Trên máy tôi thì **không**. Sinh chữ nhanh hơn 11% nhưng TTFT chậm
gấp đôi và chất lượng bắt đầu lệch; máy có 15 GB RAM nên tiết kiệm 0.73 GB cũng không quan trọng.
Tôi giữ bản 4-bit, chỉ dùng 2-bit khi máy có 4–8 GB RAM không chứa nổi bản 4-bit.
