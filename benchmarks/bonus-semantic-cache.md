# Bonus C8 - Semantic cache

Host `Darwin-arm64` · phân tích offline bằng bag-of-words từ `semantic-cache-demo.py`

## Thiết lập

Tôi dùng logic demo offline của bonus C8 để chẩn đoán lớp semantic cache mà không
cần mở thêm port. Mục tiêu không phải là báo hit rate đẹp, mà là chứng minh hai lỗi
điển hình của một embedder yếu:

- false hit: một prompt không liên quan vẫn match vì trùng từ khóa
- false miss: một paraphrase thật lại không match vì không dùng chung token

Tôi dùng một stream ngắn tự tạo:

| # | Prompt | Best sim với các prompt trước | Kết quả @0.35 | Kết quả @0.85 |
|--:|---|--:|---|---|
| 1 | What is prefix caching? | 0.000 | miss | miss |
| 2 | What is the prefix of a DNA sequence? | 0.408 | HIT | miss |
| 3 | Explain TTFT and TPOT. | 0.000 | miss | miss |
| 4 | What does time to first token mean? | 0.000 | miss | miss |

## Kết quả

- False hit của tôi là prompt #2 ở similarity `0.408`.
- False miss của tôi là prompt #4 ở similarity `0.000`.

Nếu đặt threshold `0.35`, prompt #2 trở thành hit sai nhưng prompt #4 vẫn miss.
Nếu tăng threshold lên `0.85`, false hit biến mất nhưng false miss vẫn còn nguyên.
Vì prompt #4 có similarity `0.000`, không có threshold dương nào cứu được nó; nếu hạ
threshold xuống `0.0`, prompt #2 lại bị hit sai. Nói cách khác, **không tồn tại một
threshold duy nhất** vừa chặn false hit vừa cứu được false miss này.

## Diễn giải

Đây chính là lý do semantic cache cần một embedding model chuyên dụng thay vì dùng
decoder state của chat model ở pooling mode. Bag-of-words hoặc mean-pooled decoder
state chỉ phản ánh trùng token, không hiểu paraphrase xa.

Điểm đáng chú ý hơn là cache tầng 1 này tiết kiệm toàn bộ compute khi hit, nhưng chỉ
đáng tin nếu embedding đủ tốt. Với bộ demo này, điều cần chẩn đoán không phải hit rate
cao hay thấp, mà là khoảng cách giữa false hit và false miss.

## Kết luận

Semantic cache hợp lý như một tầng trên cùng, nhưng chỉ khi:

- có embedding model đủ mạnh
- threshold được đo trên dữ liệu thật của ứng dụng
- cache được salt theo tenant để tránh leakage qua timing

Trên máy này, offline demo đủ để chứng minh rằng threshold không thể tự sửa một
embedder yếu.
