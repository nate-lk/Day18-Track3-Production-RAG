# Individual Reflection — Lab 18

**Tên:** Lưu Lê Gia Bảo
**Module phụ trách:** M4 — RAGAS Evaluation

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** `src/m4_eval.py`
- **Các hàm/class chính đã viết:**
  - `evaluate_ragas()` — `Dataset.from_dict({question, answer, contexts, ground_truth})` → `ragas.evaluate(dataset, metrics=[...])` → `result.to_pandas()` → extract per-question `EvalResult` → trả về dict với 4 aggregate scores + `per_question` list.
  - `failure_analysis()` — tính `avg_score = mean(4 metrics)` mỗi câu → sort ascending → bottom-N → map `worst_metric` vào Diagnostic Tree (`faithfulness<0.85` → "LLM hallucinating", v.v.) → trả về list `{question, worst_metric, score, diagnosis, suggested_fix}`.
  - `save_report()` — serialize ra `ragas_report.json` với timestamp, aggregate scores, failure list.
  - `load_test_set()` — đọc `test_set.json`, validate format.
- **Số tests pass:** 4/4

---

## 2. Kiến thức học được

- **Khái niệm mới nhất:** RAGAS 4 metrics và ý nghĩa từng cái: Faithfulness (LLM không hallucinate), Answer Relevancy (câu trả lời đúng câu hỏi), Context Precision (chunk retrieved có liên quan), Context Recall (đủ thông tin để trả lời). Mỗi metric chỉ ra vấn đề ở layer khác nhau của pipeline.
- **Điều bất ngờ nhất:** Context Precision và Context Recall có thể trade-off nhau — retrieve nhiều chunks thì recall tăng nhưng precision giảm. Reranking giải quyết bằng cách giữ recall của search nhưng tăng precision của context window đưa vào LLM.
- **Kết nối với bài giảng:** Slide "RAG Failure Modes" và Error Tree — giờ nhìn vào RAGAS score là biết ngay vấn đề ở đâu: faithfulness thấp → LLM issue, context_recall thấp → retrieval issue, context_precision thấp → ranking issue.

---

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** RAGAS yêu cầu `contexts` phải là `list[list[str]]` (list of lists, mỗi phần tử là list contexts cho 1 câu hỏi), nhưng pipeline trả về `list[str]`. Type mismatch → RAGAS chạy xong nhưng trả về `NaN` cho mọi score, không raise exception.
- **Cách giải quyết:** Đọc RAGAS source code, phát hiện format requirement. Fix: `contexts_nested = [[ctx] if isinstance(ctx, str) else ctx for ctx in contexts]`. Thêm validation bước đầu kiểm tra type trước khi pass vào RAGAS.
- **Thời gian debug:** ~35 phút chỉ cho bug này. Lesson: validate input/output format trước khi integrate với external libraries.

---

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Lưu per-question score ra report thay vì chỉ aggregate mean — sẽ giúp failure analysis chi tiết và dễ visualize score distribution. Thêm p25/p75 percentile vào aggregate stats.
- **Module nào muốn thử tiếp:** M1 (Chunking) — sau khi thấy context_recall thấp ở một số câu hỏi về cross-reference điều khoản trong Nghị định, muốn thử tune hierarchical chunking để tăng recall.

---

## 5. Tự đánh giá

| Tiêu chí        | Tự chấm (1-5) |
| --------------- | ------------- |
| Hiểu bài giảng  | 5             |
| Code quality    | 4             |
| Teamwork        | 5             |
| Problem solving | 4             |
