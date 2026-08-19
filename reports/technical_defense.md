# Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Trần Bảo Phúc
**MSSV:** 2A202601148
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 19/08/2026

---

## 1. Coreference Resolution
> Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

- **Ví dụ từ dữ liệu:** chunk `1a05beb7aa3071be6fd7::c0000` (bài "onsemi and Sineng Electric Spearhead the Development of Sustainable Energy Applications") có văn bản gốc dạng: *"(Nasdaq: ON) a leader in intelligent power and sensing technologies today announced that Sineng Electric will integrate onsemi EliteSiC silic..."* — câu chứa 2 thực thể công ty (onsemi, Sineng Electric) xuất hiện gần nhau, và các câu tiếp theo trong bài nhiều khả năng dùng đại từ chung chung như "the company"/"it" để nhắc lại.
- **Hiện tượng:** với quy tắc conservative ("chỉ resolve khi antecedent rõ trong cùng chunk"), khi 1 chunk có **2 công ty** xuất hiện gần nhau, mô hình có rủi ro gán "the company"/"it" nhầm sang công ty được nhắc **gần nhất** về mặt vị trí câu chữ thay vì công ty là chủ ngữ chính của hành động (ví dụ nhầm "the company" là Sineng Electric thay vì onsemi, hoặc ngược lại).
- **Hậu quả đối với Graph:** nếu resolve sai chủ ngữ, bước NER+RE ở Module 2 sẽ gán nhầm quan hệ (vd. `DEVELOPED`/`USES`) cho đúng loại thực thể nhưng sai chiều/sai chủ thể → tạo False Edge trong Neo4j, làm sai lệch kết quả truy vết đồ thị cho các câu hỏi liên quan tới 2 công ty này sau này.
- *(Lưu ý: đây là ví dụ minh hoạ dựa trên cấu trúc văn bản thật của chunk trên; để có bằng chứng resolve sai 100% chính xác, cần đối chiếu trực tiếp `text` gốc và `resolved_text` qua `coref_df` — lệnh kiểm tra nhanh: `coref_df[coref_df.unresolved_mentions.map(len) > 0].head(5)`.)*

---

## 2. Entity Resolution Threshold
> Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Vì sao chọn ngưỡng đó?

- **Threshold đã chọn:** `0.90` (tham số `threshold=0.90` trong `build_resolution_map()`), kèm Lexical Guard với `SequenceMatcher.ratio() >= 0.72`.
- **Lý do:** 0.90 là ngưỡng cao, ưu tiên precision hơn recall — tránh gộp nhầm các thực thể chỉ "gần giống" về mặt embedding (ví dụ tên người trùng họ, hoặc tên sản phẩm chứa tên công ty). Lexical Guard là lớp chặn thứ 2, đảm bảo dù vector similarity cao vẫn phải vượt qua kiểm tra chuỗi ký tự (sau khi bỏ hậu tố công ty) mới được merge.

---

## 3. Lexical Guard chặn False Merge
> Trích dẫn 1 cặp thực thể có độ tương đồng vector cao (> 0.85) nhưng bị Lexical Guard chặn không cho gộp. Giải thích lý do.

**Thực tế phát sinh trong lần chạy này:** bảng `entity_resolution_audit_df` chỉ có đúng **1 dòng**, và đó là một ca `MERGE_VECTOR` được **chấp nhận** (không phải bị từ chối):

| type | left | right | similarity | decision |
|---|---|---|---|---|
| Technology | X90A | X90 | 0.9206 | MERGE_VECTOR |

Lọc riêng `decision == "REJECT_GUARD"` trả về **DataFrame rỗng** — ở quy mô dữ liệu của lab này (`EXTRACTION_MAX_CHUNKS=400`, dataset chỉ có `description` ngắn), số lượng cặp thực thể gần giống nhau đủ để kích hoạt vector search (>0.90) rất ít, nên chưa phát sinh ca nào bị Lexical Guard chặn thật.

**Để có ví dụ minh chứng guard hoạt động đúng**, có thể chạy kiểm thử thủ công (không phải từ audit pipeline, mà gọi trực tiếp hàm guard để chứng minh logic đúng):
```python
print(merge_guard("Apple", "Apple Watch"))      # kỳ vọng False
print(merge_guard("Sam Altman", "Steve Altman")) # kỳ vọng False
```
Nếu muốn ca REJECT_GUARD phát sinh tự nhiên từ audit thật, cần tăng `EXTRACTION_MAX_CHUNKS` hoặc hạ ngưỡng threshold để có nhiều cặp ứng viên hơn.

---

## 4. Super-node Analysis
> Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì?

| Hạng | Tên thực thể | Loại (Type) | Degree |
|------|--------------|-------------|--------|
| 1 | Claroty | Company | 3 |
| 2 | FacctViewTM | Technology | 2 |
| 3 | Ridgewood Infrastructure | Company | 2 |

(Từ `test_supernode_policy()` + `graph_checks()`, đồ thị có 109 node / 62 edge.)

**Nhận xét quan trọng:** degree cao nhất thực tế chỉ là **3**, thấp hơn rất nhiều so với ngưỡng `SUPER_NODE_DEGREE=100`. Ở quy mô lab (400 chunk trích xuất, `graph_supernode_events=0` ở mọi câu hỏi Golden Eval), cơ chế Super-node Mitigation **chưa từng được kích hoạt trong thực tế** — không có super-node thật nào xuất hiện. Cơ chế đã được implement và verify đúng qua `test_supernode_policy()` (chạy không lỗi ở nhánh "không phải super-node"), nhưng chưa được stress-test ở nhánh degree > 100 vì dữ liệu quá nhỏ.

- **Ưu điểm của Temporal Mitigation (ORDER BY published_date DESC LIMIT 50):** khi graph mở rộng ở quy mô lớn hơn (ví dụ scale 350MB), các công ty lớn như Google/Microsoft chắc chắn sẽ có degree > 100; ưu tiên cạnh mới nhất giữ ngữ cảnh liên quan tới câu hỏi thời sự, tránh bùng nổ context/token.
- **Rủi ro:** nếu câu hỏi liên quan sự kiện lịch sử xa hơn (ví dụ "công ty X mua lại ai vào 2019"), cạnh cũ có thể bị cắt mất dù vẫn liên quan, dẫn tới GraphRAG trả lời thiếu.

---

## 5. Bảng so sánh Benchmark (LLM-as-a-Judge, 25 câu Golden Dataset)

| Nhóm câu hỏi | Metric | Flat RAG | GraphRAG | Δ | Nhận xét |
|---|---|---|---|---|---|
| cross-doc | Comprehensiveness | 3.091 | 2.727 | -0.364 | Flat RAG tốt hơn |
| cross-doc | Faithfulness | 3.455 | 3.091 | -0.364 | Flat RAG tốt hơn |
| cross-doc | Multi-hop reasoning | 3.091 | 2.727 | -0.364 | Flat RAG tốt hơn |
| cross-doc | Latency (s) | 2.197 | 1.839 | -0.358 | GraphRAG nhanh hơn |
| cross-doc | Token usage | 652.1 | 544.7 | -107.4 | GraphRAG rẻ hơn |
| factoid | Comprehensiveness | 3.000 | 3.000 | 0 | Ngang nhau |
| factoid | Faithfulness | 3.000 | 3.000 | 0 | Ngang nhau |
| factoid | Multi-hop reasoning | 3.000 | 3.000 | 0 | Ngang nhau |
| factoid | Latency (s) | 1.193 | 2.707 | +1.514 | Flat RAG nhanh hơn hẳn |
| factoid | Token usage | 615.5 | 503.0 | -112.5 | GraphRAG rẻ hơn |
| multi-hop | Comprehensiveness | 2.250 | 2.583 | +0.333 | GraphRAG tốt hơn |
| multi-hop | Faithfulness | 2.500 | 3.000 | +0.500 | GraphRAG tốt hơn |
| multi-hop | Multi-hop reasoning | 2.250 | 2.667 | +0.417 | GraphRAG tốt hơn |
| multi-hop | Latency (s) | 2.473 | 2.341 | -0.132 | Ngang nhau |
| multi-hop | Token usage | 684.7 | 601.8 | -82.9 | GraphRAG rẻ hơn |

**Phân tích:** GraphRAG chỉ thắng rõ ở nhóm **multi-hop** (đúng với kỳ vọng — đây là loại câu hỏi cần nối nhiều quan hệ), nhưng lại **thua** Flat RAG ở nhóm **cross-doc**. Lý do khả dĩ: (1) dataset chỉ có `description` ngắn thay vì full-text bài báo, nên đồ thị trích xuất được thiếu chi tiết để tổng hợp thông tin xuyên nhiều tài liệu; (2) sau khi đổi provider sinh câu trả lời sang OpenAI (mục 7), cả 2 nhánh dùng chung 1 model — khác biệt chất lượng chỉ còn phụ thuộc vào context (đồ thị vs vector), không phải năng lực model.

*(2 ca lỗi điển hình được phân tích chi tiết trong `failure_analysis.md`.)*

---

## 6. Trade-offs Quality vs Cost vs Latency
> So sánh đánh đổi giữa GraphRAG vs Flat RAG.

Theo bảng trên, GraphRAG **rẻ hơn về token** ở cả 3 nhóm (context đồ thị được nén/linearize gọn hơn top-k chunk thô của Flat RAG), nhưng **không nhanh hơn đáng kể** (GraphRAG cần thêm bước seed extraction + graph traversal trước khi sinh câu trả lời, cộng dồn latency; thấy rõ nhất ở nhóm factoid, GraphRAG chậm hơn Flat RAG hơn gấp đôi vì với câu hỏi đơn giản, chi phí traversal đồ thị không mang lại giá trị tương xứng). Về chất lượng, GraphRAG chỉ chứng minh được lợi thế ở đúng nhóm nó được thiết kế cho (multi-hop).

---

## 7. Quyết định từ chối/điều chỉnh đề xuất của AI Coding Agent
> Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn từ chối áp dụng? Tại sao?

- **Bối cảnh:** Trong lúc chạy Golden Evaluation, Groq free-tier bị cạn hạn mức 200.000 token/ngày (TPD) do bước NER+RE extraction (400 chunk) đã tiêu tốn gần hết ngân sách trước khi evaluation chạy tới.
- **Đề xuất của AI Coding Agent:** để tiết kiệm chi phí/thời gian, chỉ nên chạy **6/25 câu** Golden Dataset (2 câu mỗi nhóm) sau khi đổi provider sinh câu trả lời từ Groq sang OpenAI.
- **Quyết định của tôi:** **không áp dụng đề xuất rút gọn** — sau khi đã reroute cả `generate_answer` và `extract_seeds` sang OpenAI (không còn phụ thuộc Groq daily quota), tôi chọn chạy **đủ toàn bộ 25 câu** để có kết quả benchmark đại diện hơn, chấp nhận đánh đổi thêm chi phí/thời gian gọi OpenAI để đổi lấy độ tin cậy thống kê cao hơn cho báo cáo.
- **Một quyết định kỹ thuật khác đã áp dụng theo đề xuất của Agent:** đổi provider sinh câu trả lời (`generate_answer`, `extract_seeds`) từ Groq sang OpenAI ngay trong pipeline — đây là quyết định kỹ thuật thực tế do giới hạn hạ tầng free-tier, không phải vì Groq kém hơn.

---

## 8. Giải pháp Scale lên 350MB (~100.000 bài báo)
> Bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

- **Bottleneck quan sát được thực tế trong lab (bằng chứng thật, không phải suy đoán):** ở quy mô nhỏ nhất theo Scale Guard (`EXTRACTION_MAX_CHUNKS=400`), riêng bước NER+RE extraction + Coreference đã tiêu gần hết hạn mức 200.000 token/ngày của Groq free tier. Đây chính là bottleneck đầu tiên khi scale lên 350MB (~100.000 bài): **số lượng lệnh gọi LLM cho extraction tăng tuyến tính theo số chunk**, và rate limit/token quota của LLM provider sẽ chặn đứng pipeline rất sớm.
- **Giải pháp kiến trúc:**
  1. Chuyển extraction sang **batch/async worker queue** với nhiều API key luân phiên hợp lệ (theo hạn mức từng nhà cung cấp, không phải né rate limit bằng nhiều tài khoản free trùng tổ chức) hoặc dùng model tự host (vd. Llama local qua vLLM) cho bước extraction khối lượng lớn, chỉ dùng API trả phí cho các bước cần độ chính xác cao (Judge).
  2. Entity Resolution: thay vector ANN full pairwise bằng **blocking theo prefix/type** trước khi so cosine, giảm số lần gọi embedding.
  3. Ingestion: giữ nguyên `UNWIND` batch 1000 — đã đúng hướng, chỉ cần tăng batch song song (multi-connection Neo4j driver).
  4. Cân nhắc **sampling có chiến lược** (theo công ty/theo thời gian) thay vì random uniform, để giữ tính đại diện mà vẫn giới hạn được số chunk cần extract.

---

## 9. Bulk Ingestion & Provenance Integrity
> Kết quả sanity check `invalid_provenance_edges` của bạn là bao nhiêu?

- **Kết quả sanity check thực tế:** `{'nodes': 109, 'edges': 62, 'invalid_provenance_edges': 0}` — **0 cạnh thiếu provenance**, đạt yêu cầu bắt buộc của lab.
- **Giải thích:** `bulk_insert_edges()` dùng `UNWIND $rows AS row` theo batch 1000 thay vì `CREATE`/`MERGE` từng dòng, và mỗi row luôn đính kèm đủ `source_chunk_id`, `published_date`, `evidence`, `confidence` ngay từ lúc tạo trong `run_extraction()` — không có đường nào tạo edge thiếu field bắt buộc, nên sanity check trả về 0 là kết quả tất yếu của thiết kế schema, không phải may mắn.
