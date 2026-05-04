# Group Report — Lab 18: Production RAG Pipeline

**Nhóm:** Production RAG Team  
**Ngày:** 04/05/2026  
**Corpus:** 3 tài liệu tiếng Việt (Nghị định 13/2023, Chính sách nhân sự, BCTC 2023)  
**Test set:** 20 câu hỏi

---

## Thành viên & Module

| Tên | Module | Nội dung | Hoàn thành | Tests pass |
|-----|--------|----------|:----------:|:----------:|
| Lê Ngọc Hải | M1: Advanced Chunking | Semantic, Hierarchical, Structure-aware chunking | ✅ | 8/8 |
| Thái Doãn Minh Hải | M2: Hybrid Search | BM25 (underthesea) + Dense (bge-m3) + RRF | ✅ | 5/5 |
| Khương Hải Lâm | M3: Reranking | CrossEncoder bge-reranker-v2-m3 + latency benchmark | ✅ | 5/5 |
| Lưu Lê Gia Bảo | M4: RAGAS Evaluation | 4 metrics + Diagnostic Tree failure analysis | ✅ | 4/4 |
| Lương Trung Kiên | M5: Enrichment | Contextual prepend + HyQA + metadata extraction | ✅ | — |

---

## Kết quả RAGAS

| Metric | Naive Baseline | Production | Δ | Đạt ngưỡng 0.75? |
|--------|:--------------:|:----------:|:-:|:----------------:|
| Faithfulness | 0.61 | **0.87** | +0.26 | ✅ |
| Answer Relevancy | 0.57 | **0.81** | +0.24 | ✅ |
| Context Precision | 0.53 | **0.79** | +0.26 | ✅ |
| Context Recall | 0.59 | **0.76** | +0.17 | ✅ |
| **Trung bình** | **0.575** | **0.8075** | **+0.2325** | ✅ |

> Pipeline: Hierarchical Chunking (M1) + Hybrid Search BM25+Dense+RRF (M2) + Cross-encoder Reranking (M3) + Enrichment contextual+hyqa (M5) + GPT-4o-mini generation

---

## Latency Breakdown

| Bước | Thời gian (ước tính) |
|------|---------------------|
| Chunking + Indexing (1 lần) | ~8s |
| BM25 search (mỗi query) | < 10ms |
| Dense search — bge-m3 embed query | ~120ms |
| RRF fusion | < 1ms |
| Cross-encoder rerank top-20 → top-3 | ~180ms |
| LLM generate (GPT-4o-mini) | ~800ms |
| **Tổng mỗi query** | **~1.1s** |

---

## Key Findings

### 1. Biggest improvement: Faithfulness (+0.26)

Cải thiện lớn nhất đến từ việc kết hợp **Cross-encoder Reranking (M3)** và **Enrichment (M5)**. Khi naive baseline dùng dense-only search, LLM nhận context chứa nhiều đoạn nhiễu → hallucinate. Sau khi thêm reranking lọc top-20 → top-3 relevant chunks, LLM chỉ nhìn thấy đúng thông tin → faithfulness tăng từ 0.61 lên 0.87.

### 2. Biggest challenge: Context Recall với câu hỏi pháp lý

Context Recall cải thiện ít nhất (+0.17, đạt 0.76 — vừa qua ngưỡng). Nguyên nhân: câu hỏi dùng ngôn ngữ thông thường trong khi tài liệu Nghị định 13/2023 dùng thuật ngữ pháp lý chuyên biệt. BM25 (dù có underthesea segmentation) vẫn bị vocabulary mismatch, ví dụ: "chuyển ra nước ngoài" vs "chuyển dữ liệu xuyên biên giới". Dense search giúp nhưng chưa đủ.

### 3. Surprise finding: Enrichment (M5) giúp Context Precision hơn Context Recall

Ban đầu kỳ vọng M5 (HyQA + contextual prepend) sẽ cải thiện Context Recall. Thực tế, contextual prepend giúp cross-encoder score chính xác hơn → Context Precision tăng mạnh (+0.26). HyQA tạo ra câu hỏi giả định phù hợp với query → embedding gần hơn → dense search ít miss hơn. Tuy nhiên với câu hỏi liệt kê nhiều điều khoản, enrichment không bù được việc chunk quá nhỏ (CHILD_SIZE=256).

---

## Failure Summary

Xem chi tiết tại `analysis/failure_analysis.md`. Tổng kết 5 failures:

| # | Question (tóm tắt) | Worst metric | Score | Root cause |
|---|-------------------|:------------:|:-----:|------------|
| 1 | Chuyển dữ liệu ra nước ngoài (Điều 25) | context_recall | 0.21 | Vocabulary mismatch — query ≠ thuật ngữ pháp lý |
| 2 | Nghĩa vụ khi lộ lọt dữ liệu (Điều 23) | faithfulness | 0.41 | LLM hallucinate thêm nghĩa vụ ngoài context |
| 3 | Kiểm soát vs xử lý dữ liệu (Điều 2) | context_recall | 0.44 | top-k=3 không đủ bao phủ toàn bộ Điều 2 |
| 4 | Dữ liệu nhạy cảm (Điều 2 khoản 3) | context_recall | 0.52 | CHILD_SIZE=256 cắt giữa danh sách 10 mục |
| 5 | Tổng tài sản BCTC | context_precision | 0.29 | Noise từ BCKQKD và BCLCTT — thiếu metadata filter |

---

## Presentation Notes (5 phút)

### 1. RAGAS Scores — Naive vs Production

| | Naive | Production | Δ |
|-|:-----:|:----------:|:-:|
| Faithfulness | 0.61 | 0.87 | **+0.26** |
| Answer Relevancy | 0.57 | 0.81 | **+0.24** |
| Context Precision | 0.53 | 0.79 | **+0.26** |
| Context Recall | 0.59 | 0.76 | **+0.17** |

### 2. Biggest Win — M3 Cross-encoder Reranking

Module M3 (Khương Hải Lâm) tạo ra impact lớn nhất. Naive baseline không có reranking → LLM nhận cả top-10 chunks lộn xộn → faithfulness 0.61. Sau khi cross-encoder lọc top-20 → top-3 bằng bge-reranker-v2-m3, LLM chỉ nhận 3 chunks thực sự relevant → faithfulness nhảy lên 0.87. Latency cost: ~180ms — chấp nhận được cho production.

### 3. Case Study — Failure #1 (Error Tree)

**Q:** "Tổ chức nước ngoài chuyển dữ liệu ra khỏi Việt Nam cần điều kiện gì?"

```
Output sai → Context thiếu (recall 0.21) → Query mismatch
→ "chuyển ra khỏi VN" ≠ "chuyển dữ liệu xuyên biên giới"
→ Fix: Query expansion + HyDE embedding
```

### 4. Next Optimization (nếu có thêm 1 giờ)

Implement **query expansion với synonym dictionary pháp lý** — tự động mở rộng query trước khi search:
- `"chuyển ra nước ngoài"` → `"chuyển dữ liệu xuyên biên giới"` 
- `"lộ lọt dữ liệu"` → `"vi phạm bảo vệ dữ liệu cá nhân"`
- `"tổng tài sản"` + filter `section=balance_sheet`

Kỳ vọng Context Recall tăng từ 0.76 → 0.85+, đưa tất cả metrics vượt 0.80.
