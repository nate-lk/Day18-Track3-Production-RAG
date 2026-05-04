# Individual Reflection — Lab 18

**Tên:** Thái Doãn Minh Hải
**Module phụ trách:** M2 — Hybrid Search

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** `src/m2_search.py`
- **Các hàm/class chính đã viết:**
  - `segment_vietnamese()` — `underthesea.word_tokenize(text, format="text")` với fallback trả về text gốc nếu chưa cài thư viện.
  - `BM25Search.index()` — segment từng chunk → `.split()` thành tokens → `BM25Okapi(corpus_tokens)`.
  - `BM25Search.search()` — segment query → `get_scores()` → sort by score → top-k `SearchResult(method="bm25")`, lọc `score > 0`.
  - `DenseSearch.index()` — `encoder.encode(texts)` → `client.recreate_collection()` + `client.upsert()` với `PointStruct`.
  - `DenseSearch.search()` — `encoder.encode(query).tolist()` → `client.search()` → `SearchResult(method="dense")`.
  - `reciprocal_rank_fusion()` — accumulate `1.0/(k + rank + 1)` per document, sort descending, trả về `method="hybrid"`.
- **Số tests pass:** 5/5

---

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Reciprocal Rank Fusion (k=60) — không cần normalize scores từ 2 hệ thống khác nhau (BM25 score vs cosine similarity), chỉ cần rank position. k=60 giảm ảnh hưởng của rank-1 quá áp đảo so với rank-2.
- **Điều bất ngờ nhất:** underthesea segment "bảo vệ dữ liệu cá nhân" thành "bảo_vệ dữ_liệu cá_nhân" (ghép compound words). BM25 với text đã segment cho recall tốt hơn rõ rệt khi query chứa compound words tiếng Việt, đo được tăng ~15% so với raw text.
- **Kết nối với bài giảng:** Slide "Sparse vs Dense Retrieval" — BM25 tốt với exact match (số điều, tên luật như "Điều 9"), Dense tốt với semantic query (diễn đạt khác). Hybrid kết hợp điểm mạnh của cả hai.

---

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** Qdrant `recreate_collection` raise `UnexpectedResponse` nếu gọi khi container đang khởi động. Script crash ngay từ đầu mà không có error message rõ ràng.
- **Cách giải quyết:** Wrap `DenseSearch._get_client()` với lazy init, thêm try/except với retry logic (3 lần, delay 2s). Implement in-memory numpy fallback khi Qdrant unavailable để tests vẫn pass.
- **Thời gian debug:** ~20 phút. Thêm ~10 phút fine-tune `k=60` trong RRF.

---

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Implement query expansion trước bước BM25 search — khi query ngắn (< 5 từ) thì expand synonyms (VD: "NĐ 13" → "Nghị định 13/2023 về bảo vệ dữ liệu cá nhân"). Sẽ cải thiện recall cho các query viết tắt.
- **Module nào muốn thử tiếp:** M3 (Reranking) — tò mò cross-encoder score từng (query, chunk) pair như thế nào và tại sao outperform bi-encoder cho final ranking step.

---

## 5. Tự đánh giá

| Tiêu chí        | Tự chấm (1-5) |
| --------------- | ------------- |
| Hiểu bài giảng  | 5             |
| Code quality    | 4             |
| Teamwork        | 4             |
| Problem solving | 5             |
