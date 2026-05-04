# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Production RAG Team  
**Thành viên:** Lê Ngọc Hải (M1) · Thái Doãn Minh Hải (M2) · Khương Hải Lâm (M3) · Lưu Lê Gia Bảo (M4) · Lương Trung Kiên (M5)  
**Ngày:** 04/05/2026

---

## RAGAS Scores — Tổng quan

| Metric | Naive Baseline | Production | Δ |
|--------|:--------------:|:----------:|:-:|
| Faithfulness | 0.61 | **0.87** | +0.26 ✓ |
| Answer Relevancy | 0.57 | **0.81** | +0.24 ✓ |
| Context Precision | 0.53 | **0.79** | +0.26 ✓ |
| Context Recall | 0.59 | **0.76** | +0.17 ✓ |

> Tất cả 4 metrics đều vượt ngưỡng 0.75. Cải thiện lớn nhất ở **Faithfulness (+0.26)** và **Context Precision (+0.26)**.

---

## Bottom-5 Failures

### #1 — Worst (avg_score: 0.48)

- **Question:** Tổ chức nước ngoài chuyển dữ liệu cá nhân ra khỏi Việt Nam cần đáp ứng những điều kiện gì?
- **Expected (ground truth):** Theo Điều 25 Nghị định 13/2023, điều kiện gồm: sự đồng ý của chủ thể dữ liệu; đánh giá tác động chuyển dữ liệu ra nước ngoài; mức bảo vệ dữ liệu tương đương Việt Nam; thông báo trước cho Bộ Công an; lưu hồ sơ chuyển dữ liệu ít nhất 5 năm.
- **Got:** Câu trả lời thiếu điều kiện "thông báo cho Bộ Công an" và "lưu hồ sơ 5 năm" — hai điểm này nằm ở Điều 25 khoản 1(d) nhưng không được retrieve.
- **Worst metric:** `context_recall = 0.21`
- **Error Tree:**
  ```
  Output sai?
  └─ Có → Context đúng không?
           └─ Không (context_recall 0.21) → Query đúng không?
                    └─ Không → Root cause: Từ khóa mismatch
                               Query dùng "chuyển ra khỏi Việt Nam"
                               Tài liệu dùng "chuyển dữ liệu xuyên biên giới"
                               → BM25 và dense đều miss Điều 25 khoản 1(d)
  ```
- **Suggested fix:** Query expansion — map synonym trước khi search:  
  `"chuyển ra khỏi Việt Nam"` → `"chuyển dữ liệu xuyên biên giới"` + `"Điều 25"`.  
  Hoặc dùng HyDE (Hypothetical Document Embeddings) để generate query embedding gần hơn với văn bản pháp lý.

---

### #2 — (avg_score: 0.52)

- **Question:** Bên kiểm soát dữ liệu có những nghĩa vụ gì khi xảy ra sự cố lộ lọt dữ liệu cá nhân?
- **Expected (ground truth):** Theo Điều 23 Nghị định 13/2023: ngăn chặn thiệt hại ngay lập tức; thông báo Bộ Công an trong 72 giờ; thông báo chủ thể dữ liệu bị ảnh hưởng; lưu hồ sơ vi phạm; phối hợp cơ quan chức năng điều tra.
- **Got:** LLM thêm nghĩa vụ "báo cáo UBND tỉnh" và "nộp phạt hành chính" không có trong context.
- **Worst metric:** `faithfulness = 0.41`
- **Error Tree:**
  ```
  Output sai?
  └─ Có → Context đúng không?
           └─ Có (context chứa đủ Điều 23) → LLM có hallucinate không?
                    └─ Có → Root cause: LLM suy diễn ngoài context
                             temperature quá cao hoặc system prompt không đủ chặt
  ```
- **Suggested fix:** Siết chặt system prompt thêm ràng buộc `"Tuyệt đối không thêm thông tin không có trong Context."` Giảm `temperature` xuống 0.0. Thêm self-consistency check: generate 3 lần, chọn câu trả lời được hỗ trợ nhiều nhất bởi context.

---

### #3 — (avg_score: 0.55)

- **Question:** Sự khác biệt giữa bên kiểm soát dữ liệu và bên xử lý dữ liệu trong Nghị định 13/2023?
- **Expected (ground truth):** Theo Điều 2 Nghị định 13/2023 — Bên kiểm soát: quyết định mục đích và phương tiện xử lý. Bên xử lý: thực hiện xử lý thay mặt bên kiểm soát qua hợp đồng. Hai bên có thể đồng nhất (Bên kiểm soát và xử lý).
- **Got:** Câu trả lời mô tả đúng bên kiểm soát nhưng bỏ qua định nghĩa "Bên kiểm soát và xử lý dữ liệu cá nhân" tại khoản 8 — vốn là điểm phân biệt quan trọng.
- **Worst metric:** `context_recall = 0.44`
- **Error Tree:**
  ```
  Output sai?
  └─ Có → Context đúng không?
           └─ Một phần (thiếu khoản 8) → Query đúng không?
                    └─ Query quá chung chung → Root cause: top-k=3 không đủ
                             Điều 2 có 8 khoản, chia thành nhiều chunks
                             chỉ retrieve được khoản 6 và 7, thiếu khoản 8
  ```
- **Suggested fix:** Tăng `TOP_K_FINAL` từ 3 lên 5 cho câu hỏi so sánh/định nghĩa. Dùng MMR (Maximal Marginal Relevance) để đảm bảo diversity, tránh retrieve 3 chunks giống nhau.

---

### #4 — (avg_score: 0.58)

- **Question:** Dữ liệu cá nhân nhạy cảm bao gồm những loại nào theo Nghị định 13/2023?
- **Expected (ground truth):** Theo Điều 2 khoản 3 Nghị định 13/2023: quan điểm chính trị/tôn giáo; hồ sơ bệnh án; nguồn gốc chủng tộc/dân tộc; đặc điểm di truyền; sinh trắc học; đời sống tình dục; dữ liệu tội phạm; thông tin khách hàng tổ chức tín dụng; dữ liệu định vị; dữ liệu trẻ em.
- **Got:** Retrieve được 7/10 loại, bỏ sót "dữ liệu định vị" và "dữ liệu trẻ em" vì hai mục này nằm cuối danh sách, bị cắt bởi `HIERARCHICAL_CHILD_SIZE=256`.
- **Worst metric:** `context_recall = 0.52`
- **Error Tree:**
  ```
  Output sai?
  └─ Có → Context đúng không?
           └─ Không đủ → Chunk bị cắt giữa danh sách?
                    └─ Có → Root cause: HIERARCHICAL_CHILD_SIZE=256 chars quá nhỏ
                             Danh sách 10 mục trong Điều 2 khoản 3 bị split thành
                             2 chunks, chunk thứ 2 không được retrieve
  ```
- **Suggested fix:** Tăng `HIERARCHICAL_CHILD_SIZE` từ 256 lên 512 chars trong `config.py`. Hoặc dùng structure-aware chunking (M1) để giữ nguyên toàn bộ danh sách liệt kê trong một chunk.

---

### #5 — (avg_score: 0.51)

- **Question:** Tổng tài sản của công ty tại thời điểm lập báo cáo tài chính là bao nhiêu?
- **Expected (ground truth):** Tổng tài sản 612.700 triệu VNĐ (cuối năm 2023), gồm tài sản ngắn hạn 194.200 triệu và tài sản dài hạn 418.500 triệu (theo Bảng cân đối kế toán tại 31/12/2023).
- **Got:** Context trả về cả 3 chunks từ `sample_03.md` (Bảng CĐKT, BCKQKD, BCLCTT) — LLM nhầm dùng số liệu từ BCKQKD (doanh thu 648.000 triệu) thay vì số tổng tài sản trên BẢNG CĐKT.
- **Worst metric:** `context_precision = 0.29`
- **Error Tree:**
  ```
  Output sai?
  └─ Có → Context có đúng không?
           └─ Có nhưng nhiễu → Quá nhiều chunks không liên quan?
                    └─ Có → Root cause: Cả 3 loại báo cáo đều match keyword "tài sản"
                             Cross-encoder không đủ mạnh để ưu tiên Bảng CĐKT
  ```
- **Suggested fix:** Thêm metadata filter theo `section: balance_sheet` trong Bảng CĐKT. Khi query chứa "tổng tài sản", ưu tiên filter `source=sample_03.md AND section=balance_sheet` trước khi rerank.

---

## Case Study — Failure #1 (Presentation)

**Question:** Tổ chức nước ngoài chuyển dữ liệu cá nhân ra khỏi Việt Nam cần đáp ứng những điều kiện gì?

**Error Tree chi tiết:**

| Bước | Câu hỏi | Kết quả |
|------|---------|---------|
| 1 | Output đúng không? | ❌ Thiếu 2/5 điều kiện tại Điều 25 |
| 2 | Context đúng không? | ❌ context_recall = 0.21 — miss Điều 25 khoản 1(d) |
| 3 | Query rewrite OK không? | ❌ "chuyển ra khỏi Việt Nam" ≠ "chuyển dữ liệu xuyên biên giới" |
| 4 | Retrieval đúng không? | ❌ BM25 + dense đều miss chunk chứa điều kiện thông báo Bộ Công an |
| 5 | **Root cause** | Vocabulary mismatch giữa query ngôn ngữ thông thường và văn bản pháp lý |

**Fix đề xuất:**
```python
# Query expansion tự động trước search
LEGAL_SYNONYMS = {
    "chuyển ra khỏi việt nam": "chuyển dữ liệu xuyên biên giới",
    "lộ lọt dữ liệu": "vi phạm bảo vệ dữ liệu cá nhân",
    "tổng tài sản": "bảng cân đối kế toán tổng tài sản",
}
```

**Nếu có thêm 1 giờ:**
- Implement HyDE: dùng LLM generate đoạn văn pháp lý giả định từ query, dùng embedding của đoạn đó để search thay vì embedding của query gốc
- Kỳ vọng context_recall tăng từ 0.21 → 0.65+ cho câu hỏi pháp lý thuật ngữ chuyên ngành
