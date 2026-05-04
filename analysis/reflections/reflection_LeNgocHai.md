# Individual Reflection — Lab 18

**Tên:** Lê Ngọc Hải
**Module phụ trách:** M1 — Advanced Chunking Strategies

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** `src/m1_chunking.py`
- **Các hàm/class chính đã viết:**
  - `chunk_semantic()` — encode từng câu bằng `all-MiniLM-L6-v2`, tính cosine similarity giữa các câu liên tiếp, tách chunk khi similarity < threshold (0.85). Đảm bảo không cắt giữa ý.
  - `chunk_hierarchical()` — tạo parent chunks (2048 chars) bằng cách gom paragraph, sau đó slide window tạo child chunks (256 chars). Mỗi child gán `parent_id` để khi retrieve child có thể lookup parent đủ context.
  - `chunk_structure_aware()` — dùng `re.split(r'(^#{1,3}\s+.+$)', text, flags=re.MULTILINE)` parse markdown headers, pair header với content, metadata chứa key `section`.
  - `compare_strategies()` — chạy cả 4 strategies, in bảng so sánh `num_chunks`, `avg_length`, `min_length`, `max_length`.
- **Số tests pass:** 8/8

---

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Hierarchical chunking với pattern retrieve-child/return-parent — đây là giải pháp elegant cho trade-off giữa embedding precision (chunk nhỏ) và context đủ rộng cho LLM (chunk lớn). Trước đây chỉ nghĩ đến fixed-size window.
- **Điều bất ngờ nhất:** Semantic chunking với văn bản pháp luật (Nghị định 13/2023) cho kết quả kém hơn structure-aware vì các điều khoản liên tiếp về chủ đề khác nhau → cosine similarity tự nhiên đã thấp → sinh ra quá nhiều micro-chunks. Phải hạ threshold từ 0.85 xuống 0.65.
- **Kết nối với bài giảng:** Slide "Chunking Trade-offs" — chunk nhỏ = precision cao, chunk lớn = recall cao. Hierarchical giải quyết cả hai: index children (nhỏ, precise), return parent cho LLM (rộng, đủ context).

---

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** `chunk_hierarchical()` đếm sai kích thước chunk với tiếng Việt vì `len(text.split())` đếm number of space-separated tokens, không phải character count. Dẫn đến parent chunk quá to hoặc quá nhỏ.
- **Cách giải quyết:** Chuyển sang đo bằng `len(text)` (character count) nhất quán với `parent_size` và `child_size` trong config (đơn vị là chars). Test lại với corpus tiếng Việt, verify children luôn nhỏ hơn parents.
- **Thời gian debug:** ~25 phút cho vấn đề này. Tổng thời gian implement: ~80 phút.

---

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Viết unit test trước (TDD style), đặc biệt test với tiếng Việt có dấu, câu hỏi chứa "?", và văn bản có table markdown. Những edge case này phát hiện muộn tốn nhiều thời gian fix.
- **Module nào muốn thử tiếp:** M2 (Hybrid Search) — muốn hiểu BM25 với tiếng Việt đã segment khác gì so với raw text và cách RRF fusion cân bằng sparse vs dense retrieval.

---

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Teamwork | 5 |
| Problem solving | 4 |
