# Bonus B1 - Prebuilt vs source build

Host `Linux-x86_64` · CPU `12th Gen Intel(R) Core(TM) i5-1235U`
Vector extensions detected: AVX2
llama.cpp `b10488` both sides · `threads=10` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 9.9 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 9.8 | 0.99x |

On this machine, **they are within 3% -- no meaningful difference**.

before: 9.9 tok/s (prebuilt release)
after:  9.8 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 0.99x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.
A gap this small usually means the prebuilt binary already dispatches to the right kernels at runtime (releases ship one libggml-cpu-*.so per microarchitecture and pick via CPUID), or that this workload is bandwidth-bound rather than instruction-bound. Both are real findings -- say which one you think it is.


## Giải thích của tôi

**Không nhanh hơn, và đó là kết quả hợp lý trên máy này — vì hai lý do độc lập.**

1. **Bản prebuilt không hề "chung chung".** Thư mục `runtime/b10488/llama-b10488/` chứa sẵn 15
   phiên bản cho từng đời CPU (`libggml-cpu-x64.so`, `-haswell`, `-skylakex`, **`-alderlake`**,
   `-zen4`, ...) và lúc chạy nó tự hỏi CPU (CPUID) rồi chọn bản phù hợp nhất. i5-1235U là đời
   Alder Lake (có AVX2 + FMA + F16C + **AVX-VNNI**, không có AVX-512), nên bản prebuilt đã dùng
   đúng `libggml-cpu-alderlake.so` — cùng bộ lệnh mà `-DGGML_NATIVE=ON` nhắm tới. Cùng commit
   mã nguồn (`9d77fa1`), cùng kernel → bản tự build không còn gì để thắng.
2. **Decode vốn bị giới hạn bởi băng thông RAM.** `make tune` cho thấy 1 thread đã đạt 78% tốc độ
   tốt nhất. Mỗi token phải đọc ~3 GB trọng số từ RAM; lệnh CPU có xịn hơn thì vẫn phải chờ RAM.

Kết quả prefill (`bonus-build-compare-pp512.md`) là nhóm đối chứng: 43.0 so với 43.6 tok/s, cũng
nằm trong sai số. Nếu bản prebuilt dùng bộ lệnh kém hơn thì prefill (phần cần sức tính) sẽ lộ ra
ngay. Để chứng minh bộ lệnh **thật sự** quan trọng khi bị thiếu, tôi build thêm các bản thiếu
lệnh cho challenge C7 — xem `bonus-c7-isa.md`.
