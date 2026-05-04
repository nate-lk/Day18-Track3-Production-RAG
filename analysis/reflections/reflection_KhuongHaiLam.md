# Individual Reflection — Lab 18

**Tên:** Khương Hải Lâm
**Module phụ trách:** M3 — Reranking

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** `src/m3_rerank.py`
- **Các hàm/class chính đã viết:**
  - `CrossEncoderReranker._load_model()` — lazy-load `BAAI/bge-reranker-v2-m3` bằng `FlagReranker(use_fp16=True)`, fallback sang `CrossEncoder` nếu `FlagEmbedding` chưa cài. Cache `self._model_type` để `rerank()` biết gọi đúng API.
  - `CrossEncoderReranker.rerank()` — tạo pairs `(query, doc["text"])` → `compute_score(pairs, normalize=True)` (FlagReranker) hoặc `predict(pairs)` (CrossEncoder) → sort descending → top-k `RerankResult` với rank index.
  - `FlashrankReranker.rerank()` — `Ranker().rerank(RerankRequest(...))`, cùng interface `RerankResult`, latency < 5ms.
  - `benchmark_reranker()` — warmup call đầu tiên (loại bỏ model load time), sau đó `n_runs` lần `time.perf_counter()`, trả về `{avg_ms, min_ms, max_ms, p95_ms}`.
- **Số tests pass:** 4/4

---

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Cross-encoder vs Bi-encoder — bi-encoder encode query và doc độc lập (fast, indexable trước), cross-encoder encode cả pair cùng lúc (slow, nhưng hiểu interaction tốt hơn). Two-stage pipeline: retrieve fast bằng ANN, rerank chính xác bằng cross-encoder. Production pattern phổ biến.
- **Điều bất ngờ nhất:** bge-reranker-v2-m3 re-rank đúng "quyền xóa dữ liệu" (Điều 9) cao hơn "quyền truy cập dữ liệu" (Điều 7) khi query là "làm thế nào để yêu cầu xóa thông tin cá nhân" — dù cả hai đều contain từ "dữ liệu cá nhân". Cross-encoder hiểu semantic intent, không chỉ keyword overlap.
- **Kết nối với bài giảng:** Slide "Two-stage Retrieval" — retrieve fast bằng ANN (top-20), rerank chính xác bằng cross-encoder (top-3). Đây là pattern production phổ biến tại các search engine lớn.

---

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** Model load lần đầu mất ~45 giây → benchmark bị tính cả thời gian load → avg latency sai lệch hoàn toàn (45000ms thay vì ~400ms).
- **Cách giải quyết:** Tách `_load_model()` thành lazy init, thêm warmup call trong `benchmark_reranker()` trước vòng lặp đo thời gian. P95 latency sau warmup: ~380ms cho 5 docs trên CPU.
- **Thời gian debug:** ~30 phút. Phần lớn thời gian đọc FlagEmbedding API docs vì thư viện ít ví dụ.

---

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Implement async batching — batch nhiều queries cùng lúc thay vì sequential. Sẽ tận dụng GPU parallelism tốt hơn và giảm latency đáng kể trong production.
- **Module nào muốn thử tiếp:** M5 (Enrichment) — muốn kiểm chứng xem contextual prepend (thêm "chunk này từ Điều X...") ảnh hưởng thế nào đến reranking score, vì chunk có context rõ ràng thì cross-encoder dễ score hơn.

---

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 5 |
| Code quality | 5 |
| Teamwork | 4 |
| Problem solving | 4 |
