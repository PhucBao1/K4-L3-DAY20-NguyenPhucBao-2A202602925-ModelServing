# Bonus B1 - Prebuilt vs source build

Host `Linux-x86_64` · CPU `12th Gen Intel(R) Core(TM) i5-1235U`
Vector extensions detected: AVX2
llama.cpp `b10488` both sides · `threads=10` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `pp512`, 3 repetitions

| Binary | Built for | pp512 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 43.0 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 43.6 | 1.01x |

On this machine, **they are within 3% -- no meaningful difference**.

before: 43.0 tok/s (prebuilt release)
after:  43.6 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.01x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.
A gap this small usually means the prebuilt binary already dispatches to the right kernels at runtime (releases ship one libggml-cpu-*.so per microarchitecture and pick via CPUID), or that this workload is bandwidth-bound rather than instruction-bound. Both are real findings -- say which one you think it is.


## Giải thích của tôi

Prefill là phần cần sức tính (xử lý cả đoạn prompt một lúc), nên nếu bộ lệnh CPU tốt hơn thì sẽ
thấy rõ ở đây — nhưng không: 43.0 so với 43.6 tok/s (1.01×, nằm trong sai số giữa các lần chạy).
Lý do: bản prebuilt tự chọn `libggml-cpu-alderlake.so` lúc chạy, bản này đã dùng
AVX2/FMA/F16C/AVX-VNNI trên i5-1235U — đúng những gì `-DGGML_NATIVE=ON` bật. Tự build chỉ có
lợi khi bản phát hành không có phiên bản cho CPU của bạn. Hiệu ứng của bộ lệnh được đo riêng
với bản build thiếu lệnh trong `bonus-c7-isa.md`.
