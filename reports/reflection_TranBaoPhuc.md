# Reflection Cá Nhân & Kế Hoạch Đồ Án — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Trần Bảo Phúc
**MSSV:** 2A202601148
**Khóa học:** AICB-K34 · Track 3: GraphRAG

---

## 1. Mapping Bài giảng vào Code

| Khái niệm trong bài giảng | Module | Hàm / Khối code cụ thể | Quan sát thực tế & đánh giá |
|---|---|---|---|
| Conservative Coreference | Module 1 | `run_coref()` / `resolve_coref_batch()` | Chạy đủ 80 batch (400 chunk / batch_size=5); có cơ chế fallback `COREF_BATCH_FAILED` khi 1 batch lỗi để không làm sập cả pipeline. |
| Schema & Allowlist Guard | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Extraction lọc đúng theo allowlist trước khi ghi triple — ví dụ mẫu thật thu được: `Sineng Electric -USES-> EliteSiC`, `Sojern -ACQUIRED-> VenueLytics`. |
| Bulk Cypher Ingestion (`UNWIND`) | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Kết quả: 109 node / 62 edge, `invalid_provenance_edges = 0` — xác nhận batch insert giữ đủ provenance cho mọi cạnh. |
| Entity Resolution & Union-Find | Module 3 | `build_resolution_map()`, class `UF` | Ở quy mô lab (400 chunk, dataset chỉ có `description` ngắn), chỉ phát sinh 1 audit row thật (`X90A`↔`X90`, similarity 0.92, MERGE_VECTOR) — cho thấy cơ chế đúng nhưng dữ liệu quá nhỏ để test đầy đủ nhánh REJECT_GUARD. |
| Super-node Degree Cap | Module 4 | `retrieve_graph_context()` / `test_supernode_policy()` | Degree cao nhất thực tế chỉ là 3 (Claroty) — thấp hơn nhiều so với ngưỡng 100, nên cơ chế cap chưa từng bị kích hoạt thật trong lần chạy này (`graph_supernode_events = 0` ở toàn bộ 25 câu Golden Eval). |
| LLM-as-a-Judge Evaluation | Module 5 | `judge_answer()` | Chạy đủ 25 câu qua `openai/gpt-4o-mini`; GraphRAG thắng rõ ở nhóm multi-hop, thua ở cross-doc — chi tiết ở `failure_analysis.md`. |

---

## 2. Quá trình Debugging & Bài học

- **Lỗi kỹ thuật phức tạp nhất gặp phải:** Groq free-tier bị cạn hạn mức 200.000 token/ngày (TPD) ngay giữa lúc chạy Golden Evaluation, do bước NER+RE extraction (400 chunk) + Coreference (400 chunk) đã tiêu gần hết ngân sách trước khi evaluation chạy tới. Vấn đề càng khó chẩn đoán vì thông báo lỗi ban đầu ("thử lại sau 3 phút") trông giống rate-limit tạm thời, nhưng thực chất là giới hạn cứng theo ngày gắn với **organization**, không phải theo API key — nên đổi key/đổi tài khoản (kể cả nhiều email khác nhau) đều không giải quyết được vì Groq gộp các lần đăng ký từ cùng thiết bị vào chung 1 tổ chức để chống lạm dụng.
- **Cách xử lý thành công:** thay vì cố né rate limit, chuyển hẳn 2 điểm gọi LLM còn lại trong luồng evaluation (`generate_answer` ở cell 3.4, `extract_seeds` ở cell 3.2) sang OpenAI — provider đã có sẵn key hoạt động ổn định cho Judge — nhờ vậy hoàn thành được đánh giá đầy đủ 25/25 câu mà không phụ thuộc Groq nữa.
- **Bài học khác trong lúc làm:**
  - Dataset HackerNoon thực tế chỉ có cột `description` (mô tả ngắn), không có full-text bài báo như code mẫu giả định (`text`/`content`/`body`) — phải tự kiểm tra schema thật bằng `pd.read_csv(..., nrows=0).columns` thay vì tin vào tên cột suy đoán.
  - Model `llama-3.3-70b-versatile` bị Groq deprecate đúng thời điểm làm lab — bài học: khi build pipeline phụ thuộc LLM provider bên thứ 3, cần kiểm tra trạng thái model trước khi launch, không hard-code giả định model sẽ luôn tồn tại.

---

## 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)

*(Placeholder — điều chỉnh lại theo đồ án thật của bạn nếu khác)*

- **Tên đồ án / dự án:** Hệ thống hỏi-đáp tri thức nội bộ (Internal Knowledge Q&A Assistant) cho dữ liệu tài liệu/báo cáo của tổ chức.
- **Đặc thù bài toán & lý do chọn giải pháp:** Phần lớn câu hỏi thực tế là tra cứu đơn lẻ (factoid) trong tài liệu, nên **Flat RAG là đủ cho baseline**; chỉ cân nhắc bổ sung GraphRAG (Hybrid) cho nhóm câu hỏi cần nối quan hệ giữa nhiều phòng ban/dự án/con người (multi-hop) — đúng bài học từ lab: GraphRAG chỉ đáng đầu tư khi có nhu cầu truy vết quan hệ rõ ràng, còn không thì overhead extraction/entity-resolution không đáng công sức bỏ ra.
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `Document`, `Person`, `Project`, `Department`
  - Relations: `AUTHORED_BY`, `BELONGS_TO`, `REFERENCES`, `ASSIGNED_TO`
- **Chiến lược xử lý Super-node & Entity Resolution:** áp dụng cùng công thức đã kiểm chứng ở lab — Manual Alias map cho tên phòng ban/dự án viết tắt, Vector ANN + Lexical Guard cho tên người trùng nhau, và Super-node cap theo `published_date`/`updated_at` mới nhất cho các node trung tâm (vd. phòng ban lớn có nhiều tài liệu tham chiếu tới).

---

## Tự đánh giá

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|---|---|---|
| Mức độ hiểu bài giảng GraphRAG | 4 | Nắm được kiến trúc, giải thích được trade-off Flat vs Graph bằng số liệu thật |
| Khả năng kiểm soát AI Coding Agent | 4 | Từ chối đề xuất rút gọn eval xuống 6 câu, giữ đủ 25 câu để kết quả đáng tin hơn |
| Chất lượng đồ thị tri thức xây dựng | 3 | Provenance sạch (0 lỗi), nhưng đồ thị nhỏ (109 node/62 edge), audit entity resolution chỉ 1 dòng |
| Khả năng phân tích và debug hệ thống | 4 | Tự xử lý được lỗi cột dữ liệu, model deprecated, rate-limit token/ngày |
