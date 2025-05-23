# Hướng dẫn hệ thống gợi ý sách

## 1. Tổng quan về hệ thống gợi ý

Hệ thống gợi ý sách của chúng tôi sử dụng phương pháp **Hybrid Recommendation**, kết hợp hai kỹ thuật nổi tiếng trong Machine Learning:
- **Content-based Filtering**: Gợi ý dựa trên nội dung và thuộc tính của sách
- **Collaborative Filtering**: Gợi ý dựa trên hành vi của người dùng tương tự

### Cách thức hoạt động

```
┌───────────────────┐       ┌─────────────────┐
│ Lịch sử đọc sách  │       │  Hồ sơ người   │
│ Đánh giá sách     │──────▶│     dùng       │
│ Sách yêu thích    │       └────────┬────────┘
└───────────────────┘                │
                                     ▼
           ┌───────────────────────────────────────┐
           │                                       │
┌──────────▼──────────┐             ┌─────────────▼──────────┐
│   Content-based     │             │    Collaborative       │
│     Filtering       │             │      Filtering         │
└──────────┬──────────┘             └─────────────┬──────────┘
           │                                       │
           │             ┌───────┐                 │
           └─────────────▶ Hybrid ◀────────────────┘
                         │ (60/40)│
                         └───┬────┘
                             │
                             ▼
                     ┌───────────────┐
                     │  Kết quả gợi  │
                     │     ý sách    │
                     └───────────────┘
```

## 2. Xây dựng hồ sơ người dùng

### Các nguồn dữ liệu

Hồ sơ người dùng được xây dựng từ ba nguồn dữ liệu chính:

| Nguồn dữ liệu     | Bảng dữ liệu          | Mô tả                                 |
|-------------------|------------------------|---------------------------------------|
| Lịch sử đọc       | `reading_history`     | Trạng thái đọc, tỷ lệ hoàn thành     |
| Đánh giá sách     | `book_reviews`        | Điểm đánh giá từ 1-5 sao              |
| Sách yêu thích    | `user_favorites`      | Danh sách sách được đánh dấu yêu thích |

### Công thức tính điểm tương tác

Mỗi cuốn sách được tính điểm tương tác theo công thức:

```
Điểm = (Điểm trạng thái × Tỷ lệ hoàn thành/10 × Số lần đọc) + (Điểm đánh giá) + (Điểm yêu thích)
```

Trong đó:
- **Điểm trạng thái**: 
  - Đã đọc xong (completed): 5 điểm
  - Đang đọc (reading): 3 điểm
  - Dự định đọc (plan_to_read): 1 điểm
  - Đã bỏ dở (dropped): 0 điểm
- **Điểm đánh giá**: (Số sao / 5) × 3
- **Điểm yêu thích**: +5 điểm nếu là sách yêu thích

## 3. Content-Based Filtering

### Nguyên lý hoạt động

Content-based filtering gợi ý sách dựa trên đặc điểm tương tự với những sách mà người dùng đã thích trong quá khứ.

### Quy trình thực hiện

1. **Xác định sở thích thể loại và tác giả**
   ```python
   # Ví dụ tính điểm ưa thích theo thể loại
   for book_id, data in user_profile.items():
       if data['category_id']:
           category_preferences[data['category_id']] += data['interaction_score']
   ```

2. **Chuẩn hóa điểm sở thích**
   ```python
   # Chuẩn hóa để có thang điểm từ 0-1
   for category_id in category_preferences:
       category_preferences[category_id] /= max_category_score
   ```

3. **Tính điểm nội dung cho sách chưa đọc**
   ```python
   # Trọng số: thể loại (60%), tác giả (40%)
   if category_id in category_preferences:
       content_score += category_preferences[category_id] * 0.6
   if author_id in author_preferences:
       content_score += author_preferences[author_id] * 0.4
   ```

4. **Kết hợp với độ phổ biến**
   ```python
   # Trọng số: nội dung (70%), độ phổ biến (30%)
   final_score = (content_score * 0.7) + (popularity_score * 0.3)
   ```

## 4. Collaborative Filtering

### Nguyên lý hoạt động

Collaborative filtering gợi ý sách dựa trên hành vi của những người dùng có sở thích tương tự.

### Quy trình thực hiện

1. **Xây dựng ma trận người dùng-sách**
   ```
   Ma trận R:      Sách_1  Sách_2  Sách_3  ...
   Người dùng_1      4.5     0       3.0
   Người dùng_2      0       5.0     4.0
   Người dùng_3      2.5     4.0     0
   ...
   ```

2. **Tính độ tương đồng cosine giữa người dùng**
   ```python
   # Sử dụng cosine_similarity từ scikit-learn
   user_similarity = cosine_similarity(user_book_matrix)
   ```

3. **Tìm người dùng tương tự nhất**
   ```python
   # Lấy 10 người dùng tương tự nhất
   similar_users = user_similarity_df[user_id].sort_values(ascending=False)[1:11]
   ```

4. **Lấy sách từ người dùng tương tự**
   ```python
   # Điểm gợi ý = độ tương đồng × điểm tương tác
   recommendations[book] += similarity * score
   ```

## 5. Hybrid Recommendation

### Nguyên lý kết hợp

Phương pháp hybrid kết hợp ưu điểm của cả hai phương pháp trên với trọng số:
- Content-based: 60%
- Collaborative: 40%

### Quy trình thực hiện

```python
# Kết hợp các gợi ý
for rec in content_recs:
    hybrid_recs[book_id]['score'] += rec['score'] * 0.6

for rec in collab_recs:
    hybrid_recs[book_id]['score'] += rec['score'] * 0.4
```

## 6. Xử lý người dùng mới

Đối với người dùng chưa có lịch sử đọc, hệ thống sẽ gợi ý sách phổ biến dựa trên:
- Xếp hạng tổng thể (`ranking_score`)
- Lượt xem cao (`views`)
- Đánh giá trung bình tốt (`avg_rating`)

```python
popular_books = books_df.sort_values('ranking_score', ascending=False).head(limit)
```

## 7. Cách tùy chỉnh hệ thống

Bạn có thể điều chỉnh các tham số trong file `config.py` để thay đổi hành vi hệ thống:

```python
RECOMMENDATION_CONFIG = {
    'content_category_weight': 0.6,    # Trọng số thể loại
    'content_author_weight': 0.4,      # Trọng số tác giả
    'content_score_weight': 0.7,       # Trọng số content-based
    'popularity_score_weight': 0.3,    # Trọng số độ phổ biến
    'hybrid_content_weight': 0.6,      # Trọng số content trong hybrid
    'hybrid_collab_weight': 0.4,       # Trọng số collaborative trong hybrid
    'default_limit': 10                # Số lượng sách gợi ý mặc định
}
```

---

Hệ thống gợi ý này áp dụng các kỹ thuật Machine Learning cơ bản nhưng hiệu quả, phù hợp cho ứng dụng đọc sách với khả năng cân bằng giữa gợi ý theo sở thích cá nhân và xu hướng phổ biến.#   B o o k - R e c o m m e n d e r -  
 