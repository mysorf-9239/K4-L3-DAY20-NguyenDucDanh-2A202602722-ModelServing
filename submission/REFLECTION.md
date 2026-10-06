# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Đức Danh **MSSV:** 2A202602722 **Cohort:** A20-K4 **Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS 26.6 (Darwin 25.6.0, arm64)
- **CPU:** Apple M2
- **Cores:** 8 / 8
- **CPU extensions:** NEON
- **RAM:** 16 GB
- **Accelerator:** Apple Metal (GPU built into the release binary, `-ngl 99`)
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=`gemma4-e2b)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (MacBook M2, không dùng Colab hay Kaggle)

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Setup chạy thẳng, chỉ vướng: Docker Desktop đang giữ cổng 8080 (image mvp project), nên `localhost:8080` trả 404 dù
llama-server đã chạy. Đổi `LAB_SERVER_PORT=8090` cho serve, smoke, load, metrics và pipeline.



---

## 2. Đo lường *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|--------------|----------:|----------:|------------------:|------------------:|---------------------:|---------------:|
| UD-Q4_K_XL   |      2.97 |      5112 |         344 / 689 |       81.9 / 94.7 |   5486 / 6431 / 6431 |           12.2 |
| UD-Q2_K_XL   |      2.24 |      5095 |         364 / 567 |       69.9 / 79.0 |   4753 / 5355 / 5355 |           14.3 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit decode nhanh hơn 1.17x (TPOT P50 69.9 so với 81.9 ms) và nhỏ hơn 0.73 GB, nhưng TTFT không đổi vì prefill thiên về
compute. Tôi hỏi cùng một câu trên cả hai server: câu giải thích chạy tốt như nhau, còn phép tính 17 x 23 thì bản 2-bit
viết 17 x 2 = 34 và mất place value, bản 4-bit viết đúng 340. Với 16 GB RAM tôi sẽ không đánh đổi chất lượng lấy 17% tốc
độ.


---

## 3. Serving under load *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users |  RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|------:|-----:|---------:|---------:|---------:|-----------------:|---------:|
|    10 | 0.30 |    23000 |    41000 |    41000 |              6.6 |        0 |
|    50 | 0.38 |    23000 |    54000 |    55000 |             10.3 |        0 |

- **Offered load tăng 5×, throughput thực tăng:** 1.27×
- **P95 tăng:** 1.32×
- **Effective concurrency ở 50 users:** 10.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.84 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hoà ngay từ khoảng 10 user. Tải tăng 5× nhưng RPS chỉ tăng 1.27× (25% của tuyến tính), trong khi effective
concurrency 10.3 vượt xa 4 slot và `/metrics` ghi `requests_deferred` tới 46. Phần P95 tăng thêm là queue time vì mọi
slot đều bận (3.84 / 4) còn compute mỗi request không đổi. Mẫu nhỏ (16 đến 22 request) nên percentile chỉ mang tính tham
khảo. Với SLO P95 dưới 30 s thì ngay 10 user đã vượt (41 s). Knob đầu tiên tôi sẽ thử là `--parallel` (2, 4, 8), so cả
RPS lẫn P95, nhưng tôi chưa chạy thí nghiệm này.


---

## 4. Integration *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day                   | Piece                         | Real hay stub? |
|-----------------------|-------------------------------|----------------|
| N16 Cloud/IaC         | không dùng                    | stub           |
| N17 Data pipeline     | không dùng                    | stub           |
| N18 Lakehouse         | không dùng                    | stub           |
| N19 Vector + features | keyword overlap trên TOY_DOCS | stub           |
| N20 Serving           | `llama-server`                | real           |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 3043.9 ms
- **stage chiếm nhiều nhất:** llm (100% của total, 3044.0 ms)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck là llm, đúng như kỳ vọng vì embed và retrieve đều là stub gần như miễn phí. Muốn giảm 2× thì tôi tấn công
decode (23 đến 30 token ở khoảng 12 tok/s, tức 1.9 đến 2.6 s trên tổng 2.7 đến 3.6 s) bằng cách giới hạn độ dài câu trả
lời, rồi tới prefill (0.8 đến 1.0 s) bằng prompt caching.


---

## 5. The single change that mattered most *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ số thread từ `-t 8` (mặc định theo số core physical) xuống `-t 1` (từ `make tune`, metric tg128)

```
before:  12.8 tok/s (-t 8)
after:   15.6 tok/s (-t 1)
speedup: 1.21×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả **khác** với kỳ vọng từ deck — nói
rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Kết quả này khác với dạng kỳ vọng (tăng tới số core physical rồi tụt): không có knee ở 8 core, đường cong chỉ đi xuống
khi thêm thread (1 thread 15.6, 4 thread 14.4, 8 thread 12.8, 16 thread 10.7 tok/s). Lý do là lần đo này chạy với
`-ngl 99`, nên toàn bộ layer nằm trên GPU Apple qua Metal và các phép nhân ma trận của decode không dùng CPU thread.
Thread CPU chỉ điều phối graph, thêm thread chỉ thêm chi phí đồng bộ và spin. Thêm nữa CPU và GPU dùng chung unified
memory của M2, nên thread thừa chiếm bớt bandwidth của đúng thứ đang giới hạn decode. Điểm 16 thread (gấp đôi số core)
chậm nhất, khớp với giải thích này.

Giới hạn: tôi mới đo tg128 với 2 lần mỗi điểm, chưa chạy lại `make bench` với `-t 1`, và chưa thử `-ngl 0` để xác nhận
trực tiếp rằng khi chạy thuần CPU thì đường cong có knee ở khoảng 8 thread.


---

## 6. Bonus *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất *(optional)*

Nhầm tưởng nhiều thread hơn thì decode nhanh hơn, nhưng thực tế triển khai trên M2 với Metal thì 1 thread lại nhanh
nhất.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
  đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI *(xem `docs/RULES.md` §3)*

Dùng Claude Code để hoàn thiện các báo cáo liên quan từ output đã có.
