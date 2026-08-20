# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 21 | 0.38 | 20000 | 30000 | 33000 | 7.3 | 0.0% |
| 50 | 21 | 0.36 | 26000 | 53000 | 58000 | 10.5 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.96x** (19% of linear) |
| P95 latency | **1.77x** |
| Effective concurrency at 50 users | 10.5 vs `--parallel 4` slots (occupancy/slot ratio 2.62) |

**Saturated.** Throughput delivered only 0.96x for 5x the offered load, and effective concurrency (10.5) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.96x while P95 moved 1.77x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server của tôi đã bắt đầu chạm saturation ngay từ mức 10 users và chắc chắn bão hoà
ở 50 users. Số khiến tôi tin nhất là `Eff. concurrency = 10.5` khi `--parallel`
chỉ có 4 slots, trong khi throughput gần như không tăng thêm từ 10 users lên 50
users (`0.38` xuống `0.36 RPS`) nhưng P95 lại nhảy từ `30000 ms` lên `53000 ms`.
Điều đó cho thấy phần tải thêm chủ yếu biến thành queue time chứ không tạo thêm
throughput thực.

Nếu muốn tăng goodput tại SLO của mình, tôi sẽ đổi `--parallel` trước tiên và tăng
nó lên mức cao hơn, vì dấu hiệu nghẽn nằm ở số slot xử lý đồng thời chứ không phải
ở model itself. Chỉ sau khi tăng slot mà P95 vẫn còn xấu, tôi mới cân nhắc các knob
khác như giảm context hoặc giảm độ dài output, vì các knob đó tác động trực tiếp
đến chất lượng và phạm vi ngữ cảnh nhiều hơn.
