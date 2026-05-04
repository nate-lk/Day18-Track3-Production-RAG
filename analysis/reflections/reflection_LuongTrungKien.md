# Individual Reflection — Lab 18

**Tên:** Lương Trung Kiên
**Module phụ trách:** M5 — Enrichment Pipeline

---

## 1. Đóng góp kỹ thuật.

- **Module đã implement:** `src/m5_enrichment.py`
- **Các hàm/class chính đã viết:**
  - `summarize_chunk()` — LLM: `gpt-4o-mini` tóm tắt 2-3 câu. Fallback extractive: lấy 2 câu đầu bằng `re.split(r'(?<=[.!?])\s+', text)`.
  - `generate_hypothesis_questions()` — LLM: generate N câu hỏi mà chunk có thể trả lời (HyQA). Fallback: tạo câu hỏi dạng "... là gì?" từ 60 ký tự đầu mỗi sentence.
  - `contextual_prepend()` — LLM: viết 1 câu context "chunk này từ tài liệu X, nói về Y" rồi prepend. Fallback: prepend tên tài liệu nếu có.
  - `extract_metadata()` — LLM: JSON `{topic, entities, category, language}`. Fallback: rule-based keyword detection cho category, regex cho entities (`Nghị định \d+/\d+`, `Điều \d+`).
  - `enrich_chunks()` — orchestrate 4 techniques, trả về `list[EnrichedChunk]` với `original_text` preserved.
- **Số tests pass:** 5/5

---

## 2. Kiến thức học được.

- **Khái niệm mới nhất:** HyQA (Hypothetical Question Answering) — thay vì chỉ index chunk text, index thêm các câu hỏi mà chunk có thể trả lời. Query của user match với hypothetical questions → bridge vocabulary gap giữa query style (conversational) và document style (formal).
- **Điều bất ngờ nhất:** Contextual prepend cải thiện reranking score rõ rệt — cross-encoder dễ score hơn khi context chunk rõ ràng "Đây là Điều 9 Nghị định 13/2023 về quyền của chủ thể dữ liệu" thay vì chunk text thuần bắt đầu giữa chừng.
- **Kết nối với bài giảng:** Slide "Contextual Retrieval" (Anthropic) — one-time indexing cost nhưng benefit mọi query sau đó. Anthropic báo cáo giảm 49% retrieval failure chỉ với contextual prepend.

---

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** Rate limit khi gọi OpenAI API để enrich ~150 chunks với tier free (60 RPM) — bị 429 error sau ~80 chunks, pipeline crash giữa chừng, mất toàn bộ kết quả đã enrich.
- **Cách giải quyết:** Implement disk cache (JSON file, keyed by `hash(text)`) để không re-enrich chunk đã xử lý. Thêm `time.sleep(1)` giữa các API calls. Với quota free tier, những chunk chưa enrich dùng extractive fallback — vẫn đảm bảo `enrich_chunks()` luôn trả về đủ kết quả.
- **Thời gian debug:** ~40 phút. Bao gồm thời gian đọc OpenAI rate limit docs và implement retry/cache logic.

---

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Async enrichment với `asyncio` + `aiohttp` và semaphore giới hạn concurrent requests. Thay vì sequential, concurrent calls (ví dụ 10 concurrent với delay) sẽ giảm tổng thời gian enrichment từ ~5 phút xuống còn ~1 phút.
- **Module nào muốn thử tiếp:** M4 (Evaluation) — muốn đo delta RAGAS score trước và sau enrichment để có số liệu quantitative, không chỉ định tính về tác động của contextual prepend và HyQA.

---

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng |  |
| Code quality | 5 |
| Teamwork | 4 |
| Problem solving | 5 |
