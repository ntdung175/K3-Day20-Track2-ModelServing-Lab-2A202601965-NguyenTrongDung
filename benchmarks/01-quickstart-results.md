# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3126 | 364 / 448 | 81.4 / 92.1 | 5477 / 5543 / 5543 | 12.3 |
| UD-Q2_K_XL | 2.24 | 4065 | 374 / 454 | 69.8 / 71.9 | 4760 / 4908 / 4908 | 14.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.16x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

UD-Q2_K_XL nhỏ hơn 0.73 GB trên đĩa và cho tốc độ decode tốt hơn trên máy của tôi:
TPOT giảm từ 81.4 ms xuống 69.8 ms, còn tốc độ decode tăng từ 12.3 lên 14.3 tok/s,
tức khoảng 1.16x nhanh hơn. TTFT gần như không đổi và ở Q2 còn hơi xấu hơn một
chút ở p50/p95, nên lợi ích chính nằm ở tốc độ sinh token chứ không phải token đầu
tiên. Nếu chỉ nhìn số đo thì bản quantization nhỏ này đáng dùng vì nhanh hơn và
chiếm ít dung lượng hơn, nhưng tôi vẫn sẽ so chất lượng câu trả lời với Q4 trước
khi dùng cho việc cần độ ổn định đầu ra cao.
