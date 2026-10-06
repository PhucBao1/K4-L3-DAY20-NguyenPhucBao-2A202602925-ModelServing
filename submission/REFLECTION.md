# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Phúc Bảo
**MSSV:** 2A202602925
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Ubuntu 24.04.4 LTS (kernel 7.0.0-34-generic)
- **CPU:** Intel Core i5-1235U (12th Gen, hybrid: 2 P-core + 8 E-core, 15 W)
- **Cores:** 10 physical / 12 logical
- **CPU extensions:** AVX2, AVX-VNNI, FMA, F16C (không có AVX-512)
- **RAM:** 15.3 GB
- **Accelerator:** CPU only (iGPU Intel UHD, không dùng; `ngl=0`)
- **llama.cpp asset đã tải:** `llama-b10488-bin-ubuntu-x64.tar.gz`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (HP Pavilion 14), không dùng Colab/Kaggle.

**Setup story** (≤ 80 chữ): `make setup` chạy một lần là xong, không lỗi: tải bản llama.cpp
dựng sẵn cho Ubuntu và 5.2 GB model trong ~2 phút. Điểm cần chú ý: CPU có core mạnh và core yếu
lẫn lộn, nên số thread mặc định (10) không phải tốt nhất — `make tune` cho thấy 5 thread nhanh
hơn. Bonus B1 cần `cmake`; máy không có nên tôi cài vào `.venv` bằng `pip install cmake`.

**Ghi chú về các lần chạy lại:** `make bench`, `make load-10` và `make load-50` được chạy **2 lần**
(lần 2 để chụp screenshot). Số trong `benchmarks/*.md` và trong report này là của **lần 2**, khớp
với ảnh. `make metrics` (`02-server-batching-u50.md`) và `make tune` là từ lần chạy đầu. Hai lần
cho cùng kết luận (vd. TTFT của Q2: 769 ms lần 1, 741 ms lần 2).

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 2041 | 338 / 571 | 90.3 / 94.9 | 5485 / 6549 / 6549 | 11.1 |
| UD-Q2_K_XL | 2.24 | 2023 | 741 / 882 | 81.2 / 82.5 | 5855 / 6032 / 6032 | 12.3 |

**Quan sát** (≤ 60 chữ): Bản 2-bit sinh chữ chỉ nhanh hơn 1.11 lần (file nhỏ hơn 25%), nhưng
TTFT chậm gấp đôi (338 → 741 ms). Hỏi cùng 3 câu: cả hai đều đúng, nhưng bản 2-bit trả năm
`"1889"` dạng chữ thay vì số. Máy có 15 GB RAM nên **không đáng** đổi; tôi giữ bản 4-bit.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.39 | 23000 | 36000 | 36000 | 7.9 | 0.0% |
| 50 | 0.36 | 19000 | 55000 | 55000 | 7.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.93× (không tăng)
- **P95 tăng:** 1.53×
- **Effective concurrency ở 50 users:** 7.9 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.85 / 4 slots

**Saturation reading** (≤ 80 chữ): Server quá tải ngay từ 10 users: tăng tải 5 lần mà RPS
không tăng (0.39 → 0.36), còn P95 tăng 1.53× (36 → 55 s). Little's Law: 7.9 request trong hệ thống
nhưng chỉ có 4 slot → ~4 đang xếp hàng; trả lời mất ~20 s thay vì 5.5 s lúc rảnh. Phần độ trễ thêm
là **xếp hàng**, không phải tính toán. Knob đầu tiên: giới hạn hàng đợi, không tăng `--parallel`.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | infra | stub (chạy local trên laptop) |
| N17 Data pipeline | ingestion | stub (doc list hard-code trong `pipeline.py`) |
| N18 Lakehouse | storage | stub (docs nằm trong memory) |
| N19 Vector + features | embed + retrieve | stub (giữ nguyên STUB 1/2, keyword overlap) |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 4588.2 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): LLM chiếm 100% vì embed/retrieve là stub. Điều bất ngờ: ngay trong
LLM, phần đọc prompt (prefill) đã chiếm 40–55% dù context chỉ 113–149 token, vì CPU chỉ đọc
được ~40–75 token/giây. Muốn nhanh gấp đôi: dùng lại phần system prompt đã tính (prefix cache),
lấy ít tài liệu hơn, dùng 8 thread cho prefill — rồi mới cắt ngắn câu trả lời.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ số thread decode từ `-t 12` (dùng hết logical cores) xuống `-t 5`

```
before:  10.6 tok/s  (tg128, -t 12; default -t 10 = 10.9 tok/s)
after:   11.4 tok/s  (tg128, -t 5)
speedup: 1.08×  (1.05× so với default -t 10; 1.31× so với -t 24)
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Khi sinh mỗi token (decode), CPU phải đọc lại gần như toàn bộ ~3 GB trọng số model từ RAM, nhưng
với mỗi byte đọc vào chỉ làm rất ít phép tính. Vì vậy nút thắt là **tốc độ đọc RAM (memory
bandwidth)**, không phải số core. Bằng chứng: chỉ 1 thread đã đạt 78% tốc độ tốt nhất (8.9 so với
11.4 tok/s), và 4–5 thread là đủ để RAM chạy hết tốc độ. Giống một cái vòi nước: vài người hứng
là vòi đã chảy hết cỡ, thêm người chỉ thêm chen lấn. Thêm thread không làm RAM nhanh hơn, chỉ
thêm thời gian các thread **chờ nhau** sau mỗi bước tính và tranh nhau bộ nhớ đệm. Riêng
i5-1235U có 2 core mạnh + 8 core yếu: llama.cpp chia việc đều cho mọi thread rồi chờ thread chậm
nhất, nên khi dùng cả core yếu thì core mạnh phải ngồi chờ. Ở 12–24 thread còn tranh core với
hệ điều hành.

Điều này **khác** slide (slide nói tốt nhất ở số core vật lý = 10; của tôi là 5). Để chắc đây là
do RAM chứ không phải đo sai, tôi đo thêm **prefill** (đọc prompt 512 token) với cùng binary:
prefill vẫn nhanh lên tới **8 thread** (20 → 43 tok/s, gấp 2.1 lần). Vì prefill xử lý cả đoạn một
lúc, mỗi byte trọng số được dùng cho nhiều token → bị giới hạn bởi sức tính, nên thêm core vẫn có
ích. Cùng model, hai loại việc, hai đường cong ngược nhau. Mức tăng decode chỉ 1.05–1.08 lần
chính là bài học: khi nút thắt là RAM, muốn nhanh hẳn phải **đọc ít byte hơn mỗi token** (model
nén nhỏ hơn) hoặc **một lần đọc phục vụ nhiều người** (continuous batching — ở §3 nó cho ~22
tok/s tổng so với ~11 tok/s khi phục vụ 1 người).

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:**
- **B1** — build llama.cpp từ mã nguồn (`-DGGML_NATIVE=ON`) và so với bản prebuilt: `benchmarks/bonus-build-compare-tg128.md`, `bonus-build-compare-pp512.md`
- **B2** — `make sweep-ctx` (độ dài prompt → chi phí prefill): `benchmarks/bonus-ctx-len-sweep.md`
- **B3** — before/after bên dưới (từ các bản build của B1/C7, không phải `make tune`)
- **B4** — **C7** khảo sát bộ lệnh CPU, 4 bản build: `benchmarks/bonus-c7-isa.md`
- **B5** — **C9** phục vụ embedding: `benchmarks/bonus-c9-embedding.md`

**Numbers:**

```
B1  prebuilt (alderlake) → native:   tg128 9.9 → 9.8 tok/s   (0.99×)   pp512 43.0 → 43.6 tok/s (1.01×)
C7  SSE2 → native (cùng mã nguồn, cùng model):
    before:  2.33 tok/s decode · 4.27 tok/s prefill   (bản chỉ có SSE2)
    after:  11.36 tok/s decode · 44.28 tok/s prefill  (bản -DGGML_NATIVE=ON)
    speedup: 4.9× decode · 10.4× prefill
```

**Điều này nói lên gì mà deck chưa nói:**

1. **"Tự build cho CPU của mình sẽ nhanh hơn" không đúng trên máy này.** Bản prebuilt chứa sẵn 15
   phiên bản cho các đời CPU và tự chọn `libggml-cpu-alderlake.so` lúc chạy — đúng bộ lệnh của
   i5-1235U. Nên build native chỉ ngang bằng (0.99–1.01×). Muốn thấy giá trị của bộ lệnh, tôi
   phải build bản **thiếu** lệnh (chỉ SSE2): chậm hơn 4.9 lần ở decode và 10.4 lần ở prefill.
   Thêm AVX-VNNI lên trên AVX2 thì không khác gì. Bài học: bậc quan trọng là AVX2; trước khi tự
   build hãy xem binary đang dùng kernel nào.
2. **Nút thắt không cố định, nó phụ thuộc vào phần mềm.** Với AVX2, decode bị giới hạn bởi RAM
   (1 thread đã đạt 78% tốc độ). Với SSE2, cùng lượng dữ liệu nhưng chậm 4.9 lần → CPU tính
   không kịp, nút thắt chuyển sang sức tính.
3. **Batching embedding trên CPU không có lợi (C9)** — ngược với deck. Gom 8–16 văn bản còn chậm
   hơn gửi 1–2 cái (1.9 so với 3.4 texts/s), và tăng slot lên 16 không đổi gì. Vì prefill đã
   dùng hết sức tính của CPU ngay ở batch 1, không còn gì để lấp đầy. Batching chỉ có lợi khi
   phần cứng còn dư (GPU) hoặc khi việc bị giới hạn bởi RAM (chat decode: batching tăng gấp đôi).
4. **Prefill chiếm phần lớn thời gian rất sớm (sweep-ctx):** ~400 token prompt là prefill đã lâu
   hơn sinh câu trả lời 64 token; 8k token phải chờ gần 4 phút mới có chữ đầu tiên. Trên CPU,
   RAG chỉ đủ chỗ cho 1–2 đoạn tài liệu ngắn.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Tôi tưởng thêm core luôn nhanh hơn, nhưng sinh token nhanh nhất ở 5 thread chứ không phải 10;
còn bản build "tối ưu cho CPU của tôi" lại không nhanh hơn bản tải về chút nào — nút thắt nằm ở
RAM, và bản tải về vốn đã chọn đúng kernel.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Claude Code (Claude Opus 5.5): chạy các lệnh `make`, đọc số liệu từ `benchmarks/*` và soạn nháp phần nhận xét/giải thích. Tôi đã đọc lại, kiểm tra số liệu với file benchmark và chỉnh sửa trước khi nộp.
