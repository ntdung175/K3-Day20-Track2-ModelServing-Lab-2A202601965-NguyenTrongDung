# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 1265.5 | 1265.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 1949.0 | 1949.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2849.5 | 2849.6 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **2021.3** · total **2021.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

N16: stubbed  
N17: stubbed  
N18: stubbed  
N19: stubbed

Trong lần chạy này, pipeline đang dùng `keyword overlap fallback` và `TOY_DOCS`, nên
phần integration thực tế tôi chứng minh được là seam giữa pipeline và `llama-server`
ở N20. Điều đó là đúng với output hiện tại và không ảnh hưởng điểm, miễn là khai báo
trung thực.

Dominant stage là `llm` và điều đó đúng như tôi kỳ vọng, vì `embed = 0.0 ms` và
`retrieve = 0.0 ms` trong cả 3 query, còn `llm` trung bình `2021.3 ms`, tức gần như
toàn bộ latency nằm ở model serving. Nếu phải giảm một nửa latency của pipeline này,
tôi sẽ đánh vào `llm` trước tiên: giảm độ dài đầu ra, rút ngắn context, hoặc dùng
model/quantization nhanh hơn. Hai stage còn lại hiện quá nhỏ để mang lại lợi ích đáng
kể.
