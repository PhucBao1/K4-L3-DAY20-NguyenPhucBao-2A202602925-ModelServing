# Bonus C7 (B4) - Bộ lệnh CPU (instruction set) ảnh hưởng thế nào

Máy: Intel i5-1235U (Alder Lake), chỉ CPU · model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · cùng mã nguồn
llama.cpp commit `9d77fa1` (b10488) cho mọi bản build · tất cả là bản **Release** (`-O3`).

CPU này hỗ trợ (từ `/proc/cpuinfo`): `avx2 fma f16c avx_vnni sha_ni` — **không có AVX-512**.

## Các bản build

| Bản | Cách build | Bộ lệnh được dùng |
|---|---|---|
| prebuilt | bản phát hành có sẵn, tự chọn `libggml-cpu-alderlake.so` lúc chạy | AVX2 + FMA + F16C + AVX-VNNI |
| native | `-DGGML_NATIVE=ON` (`-march=native`) | mọi thứ CPU có |
| avx2 | `-DGGML_NATIVE=OFF` + AVX/AVX2/FMA/F16C/BMI2 = ON, `GGML_AVX_VNNI=OFF` | như native nhưng **bỏ AVX-VNNI** |
| sse2 | `-DGGML_NATIVE=OFF` + tắt hết AVX/AVX2/FMA/F16C/BMI2/SSE4.2 | chỉ SSE2 (mức tối thiểu của mọi CPU x86-64) |

Lưu ý: `-DGGML_NATIVE=OFF` một mình **không** tạo ra bản "cơ bản" — CMake của ggml mặc định vẫn
bật AVX2/FMA/F16C (mức Haswell). Tôi phải tắt từng cờ một mới có bản SSE2 thật.

## Kết quả (`llama-bench -r 2`; decode ở `-t 5`, prefill ở `-t 8`)

| Bản | tg128 decode (tok/s) | pp512 prefill (tok/s) | decode so với sse2 | prefill so với sse2 |
|---|--:|--:|--:|--:|
| sse2 | 2.33 | 4.27 | 1.00× | 1.00× |
| avx2 (không VNNI) | 11.35 | 44.29 | 4.87× | 10.4× |
| native | 11.36 | 44.28 | 4.88× | 10.4× |
| prebuilt (alderlake) | 11.13 | 43.71 | 4.78× | 10.2× |

## Phân tích

**1. SIMD là thứ quan trọng nhất: AVX2 nhanh hơn SSE2 gấp 4.9 lần (decode) và 10.4 lần (prefill).**
SSE2 xử lý 128 bit mỗi lệnh và không có lệnh nhân-cộng gộp (FMA), lại không có các lệnh xử lý số
nguyên 8-bit 256 bit mà kernel Q4_K dùng để giải nén và nhân trọng số 4-bit. AVX2 xử lý 256 bit
mỗi lệnh + FMA → mỗi lệnh làm được nhiều gấp vài lần. Với bản SSE2, phần lớn kernel lượng tử hoá
phải rơi về code vô hướng (scalar), nên chậm hơn rất nhiều.

**2. Điều thú vị: không có SIMD thì decode KHÔNG còn bị giới hạn bởi RAM nữa.** Ở bản AVX2, decode
bị giới hạn bởi băng thông RAM (xem `make tune`: 1 thread đã đạt 78% tốc độ tốt nhất). Nhưng bản
SSE2 đọc **cùng lượng trọng số** mỗi token mà chỉ đạt 2.33 tok/s — trong khi bản AVX2 chứng minh
RAM này đọc được nhanh gấp 4.9 lần. Nghĩa là CPU **tính không kịp** để dùng hết RAM: nút thắt
chuyển từ RAM sang sức tính. Prefill vốn đã cần
sức tính nên bị ảnh hưởng nặng hơn (10.4× so với 4.9×). Cùng một model, nút thắt nằm ở đâu phụ
thuộc vào việc phần mềm có tận dụng đúng phần cứng hay không.

**3. AVX-VNNI không tạo ra khác biệt** (avx2 không VNNI 11.35 / 44.29 so với native 11.36 / 44.28).
Tôi đoán vì: (a) decode đã bị giới hạn bởi RAM, lệnh tính nhanh hơn cũng không giúp; (b) với
prefill, kernel Q4_K trong bản này có lẽ dùng đường AVX2 (`maddubs`) là chủ yếu, nên VNNI dạng 256
bit trên Alder Lake không được tận dụng nhiều. Kết luận thực tế: trên CPU này, **AVX2 là bậc
quan trọng**, còn các bậc trên nó gần như không thêm gì.

**4. Liên hệ với datacenter (FA3 cho Hopper, FA4 cho Blackwell):** "kernel phải khớp với chip" là
đúng — nhưng chỉ có lợi khi kernel cũ **thiếu** khả năng mà chip có. Ở đây bản phát hành đã tự
chọn đúng kernel (alderlake) lúc chạy, nên tự build native không thêm gì (B1: 0.99–1.01×). Còn
khi chạy sai kernel (SSE2) thì mất tới 80–90% tốc độ. Bài học: trước khi tự build, hãy kiểm tra
binary đang thực sự dùng kernel nào.
