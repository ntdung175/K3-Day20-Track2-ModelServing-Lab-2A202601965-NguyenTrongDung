# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **8 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 15.1 | 100% |
| 4 | 14.8 | 98% |
| 8 | 12.1 | 80% |
| 16 | 10.8 | 71% |

**Best**: `-t 1` at 15.1 tok/s
**Slowest tested**: `-t 16` at 10.8 tok/s (1.40x spread)
**Against the physical-core default** (`-t 8`, 12.1 tok/s): 1.25x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

Knee của đường cong nằm ngay ở `-t 1`, vì đây là điểm cho throughput cao nhất
trên máy của tôi. Tăng lên `-t 4`, `-t 8` rồi `-t 16` đều làm `tg128` giảm dần,
cho thấy thêm thread không giúp mà còn tạo overhead.

Lý do hợp lý nhất là bài đo này đang bị giới hạn bởi tranh chấp tài nguyên hơn là
thiếu số thread. Khi số thread tăng, các luồng phải cạnh tranh cho cache, bộ nhớ và
chi phí lập lịch, nên lợi ích song song bị triệt tiêu. Trên M2 của tôi, một thread
đã đủ để đẩy pipeline hiệu quả hơn, còn nhiều thread hơn chỉ làm tăng nhiễu và
giảm throughput.

Điểm đáng chú ý là đường cong không peak ở số core vật lý `8` như kỳ vọng thông
thường. Đây không phải lỗi của benchmark; ngược lại, nó cho thấy cấu hình hiện tại
không hưởng lợi từ việc scale thread và phù hợp hơn với chế độ chạy ít thread.
