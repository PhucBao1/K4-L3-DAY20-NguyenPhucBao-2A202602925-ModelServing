# Bonus - Context-length sweep (prefill cost)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=10` `ngl=0` · RAM 15.3 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 62.2 | 4117.1 | 1.00x |
| 1024 | 41.4 | 24758.2 | 1.50x |
| 2048 | 39.6 | 51691.1 | 1.57x |
| 4096 | 37.1 | 110493.7 | 1.68x |
| 8192 | 35.4 | 231151.2 | 1.75x |

At 8192 tokens, prefill costs **231151 ms** --
1.75x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Phát hiện của tôi

**Trên CPU này, prefill chiếm phần lớn thời gian từ rất sớm — khoảng 400 token prompt.** Sinh một
câu trả lời 64 token mất 64 × 93.9 ms ≈ 6.0 s. Prefill với prompt 256 token đã mất 4.1 s; ở 1,024
token mất 24.8 s — gấp 4 lần toàn bộ thời gian sinh câu trả lời. Ở 8,192 token, người dùng phải
chờ **3 phút 51 giây** mới thấy chữ đầu tiên. Trên laptop này, vấn đề của RAG là **TTFT** (chờ
chữ đầu tiên), không phải tốc độ sinh chữ.

**Có thấy đường cong bậc hai (O(N²)) không?** Có một phần, nhưng không đúng chỗ tôi nghĩ. Tốc độ
prefill giảm mạnh nhất trong khoảng 256 → 1,024 token (62 → 41 tok/s, giảm 33%), sau đó chỉ giảm
từ từ (41 → 35 tok/s dù prompt dài thêm 8 lần). Nếu O(N²) của attention là nguyên nhân chính thì
tốc độ phải giảm **ngày càng nhanh** khi prompt dài ra, chứ không chững lại. Cách tôi hiểu:

- **256 → 1,024:** prompt 256 token vừa trong một lần xử lý (mặc định `-ub 512`) và dữ liệu
  tạm vừa trong bộ nhớ đệm CPU; prompt dài hơn phải chia nhiều lần và tràn khỏi bộ nhớ đệm.
  Đây là một bậc thang một lần, không phải đường cong.
- **1,024 → 8,192:** phần giảm từ từ mới là chi phí attention thật, nhưng nhỏ — vì với model
  ~2B tham số, phép nhân với trọng số vẫn chiếm phần lớn, và nhiều lớp của Gemma chỉ nhìn một
  "cửa sổ" ngắn quanh token (sliding window), chỉ vài lớp toàn cục mới tốn N². Nên "chậm hơn
  tuyến tính 1.75×" là kích thước thật của hiệu ứng bậc hai ở đây.

**Ý nghĩa cho pipeline RAG của tôi:** prefill ~40 tok/s → mỗi đoạn tài liệu 200 token làm TTFT
tăng thêm ~5 s. Muốn TTFT dưới ~5 s thì chỉ đủ chỗ cho câu hỏi + system prompt + **tối đa 1–2
đoạn ngắn** (~300–400 token tổng). Nhồi 8k context "vì model cho phép" là không khả thi trên CPU —
các knob quan trọng là: lấy ít/ngắn hơn, rerank trước khi gửi cho LLM, và prefix cache cho phần
prompt cố định.
