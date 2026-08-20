# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Trọng Dũng
**Cohort:** K3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS (Darwin 24.6.0)
- **CPU:** Apple M2
- **Cores:** 8 physical / 8 logical
- **CPU extensions:** NEON
- **RAM:** 16.0 GB
- **Accelerator:** Apple Metal
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=` gemma4-e2b)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Chạy local trên MacBook Pro M2 16 GB, dùng model mặc định Gemma 4 E2B. `make setup`
tải runtime và model thành công nên không cần cloud fallback. Mình từng thử
`llama serve -hf` và bị rớt kết nối khi tải model, nhưng luồng chuẩn của lab là để
`make setup` ghi `models/active.json` rồi dùng `make serve`, nên không cần workaround
thêm.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3126 | 364 / 448 | 81.4 / 92.1 | 5477 / 5543 / 5543 | 12.3 |
| UD-Q2_K_XL | 2.24 | 4065 | 374 / 454 | 69.8 / 71.9 | 4760 / 4908 / 4908 | 14.3 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

UD-Q2_K_XL nhỏ hơn 0.73 GB trên đĩa và cho tốc độ decode tốt hơn trên máy của tôi:
TPOT giảm từ 81.4 ms xuống 69.8 ms, còn tốc độ decode tăng từ 12.3 lên 14.3 tok/s,
tức khoảng 1.16x nhanh hơn. TTFT gần như không đổi và ở Q2 còn hơi xấu hơn một chút
ở p50/p95, nên lợi ích chính nằm ở tốc độ sinh token chứ không phải token đầu tiên.
Tôi chưa ghi lại xong phần so sánh chất lượng bằng `--compare`, nên kết luận hiện tại
mới dựa trên timing; cần thêm một lượt hỏi cùng câu trên cả hai quantization trước
khi chốt xem Q2 có đáng dùng hơn Q4 trong thực tế hay không.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.38 | 20000 | 30000 | 33000 | 7.3 | 0.0% |
| 50 | 0.36 | 26000 | 53000 | 58000 | 10.5 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.96×
- **P95 tăng:** 1.77×
- **Effective concurrency ở 50 users:** 10.5 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.92 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server của tôi bão hoà ở mức 50 users, và dấu hiệu sớm đã bắt đầu từ 10 users:
throughput gần như không tăng thêm từ 10 lên 50 users, nhưng P95 nhảy từ 30000 ms
lên 53000 ms. Số thuyết phục nhất là `Eff. concurrency = 10.5` trong khi `--parallel`
chỉ có 4 slots, nghĩa là phần tải thêm chủ yếu biến thành queue time chứ không tạo
thêm throughput. Nếu phải tăng goodput@SLO, tôi sẽ đổi `--parallel` trước tiên vì
nghẽn nằm ở số slot xử lý đồng thời, không phải ở một bottleneck khác của pipeline.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | stubbed |
| N17 Data pipeline | stubbed |
| N18 Lakehouse | stubbed |
| N19 Vector + features | stubbed |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 2021.3 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Trong lần chạy này, bottleneck nằm gần như hoàn toàn ở `llm`, còn embed/retrieve
đều bằng 0.0 ms vì pipeline đang dùng `keyword overlap fallback` trên `TOY_DOCS`.
Điều này khớp với kỳ vọng của mình: seam của N20 đã chạy thật, nhưng phần N16-N19
vẫn đang stub nên không tạo thêm latency đáng kể. Nếu phải giảm 2x latency, tôi sẽ
tấn công vào LLM trước tiên bằng model nhỏ hơn, output ngắn hơn, hoặc context ngắn
hơn; hai stage còn lại quá nhỏ để cắt nhiều được.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ `-t` từ 8 xuống 1

```
before:  12.1 tok/s
after:   15.1 tok/s
speedup: 1.25×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Trên máy M2 của mình, `-t 1` là điểm tốt nhất cho `tg128`, còn tăng thread lên 4,
8 rồi 16 đều làm throughput giảm. Điều này nói rằng bài đo này không được giới hạn
bởi thiếu thread mà bởi overhead và tranh chấp tài nguyên: càng nhiều thread, càng
nhiều chi phí lập lịch, cache pressure và tranh chấp memory bandwidth. Vì decode
đang nghiêng về memory-bound hơn là compute-bound, thêm thread không giúp mà còn
làm chậm. Đây là một kết quả hơi ngược kỳ vọng nếu chỉ nhìn số core vật lý, nhưng
lại rất hợp với cơ chế của llama.cpp trên model này.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B5 C8 semantic cache (offline diagnostic)

**Numbers:**

```
false hit: sim 0.408 ở threshold 0.35
false miss: sim 0.000 ở threshold 0.85
speedup: n/a — đây là chẩn đoán threshold, không phải speedup
```

**Điều này nói lên gì mà deck chưa nói:**

Semantic cache chỉ hữu ích nếu embedding đủ tốt. Với embedder yếu kiểu bag-of-words
hoặc decoder-state pooling, threshold thấp sẽ kéo cả prompt không liên quan vào cache,
còn threshold cao thì vẫn bỏ sót paraphrase thật. Vì vậy, không có một ngưỡng chung nào
giải quyết được cả false hit lẫn false miss trên stream này; thứ cần chỉnh trước là
chất lượng embedding, rồi mới đến threshold.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

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
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
