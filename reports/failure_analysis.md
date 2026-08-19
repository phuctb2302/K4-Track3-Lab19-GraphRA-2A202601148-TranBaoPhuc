# Phân Tích Ca Lỗi — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Trần Bảo Phúc
**MSSV:** 2A202601148

Dữ liệu lấy từ `outputs/graphrag_eval_results.csv` (chạy đủ 25 câu Golden Dataset, LLM-as-a-Judge: `openai/gpt-4o-mini`).

---

## Ca lỗi 1 — Flat RAG thất bại, GraphRAG thành công

- **Question ID:** `G5000-49` (multi-hop)
- **Câu hỏi:** "Across the selected Samsung records, identify three distinct technology domains Samsung is connected to and the specific evidence for each."
- **Reference:** Display/biometric sensing (Sensor OLED Display), Smart home (Aqara/SmartThings/FP2 Presence Sensor), Semiconductors (System LSI Tech Day).

**Flat RAG** (điểm: Comprehensiveness=1, Faithfulness=1, Multi-hop=1):
> Trả lời chung chung: "Information Technology", "Semiconductors", "Wireless Telecommunications" — không khớp bất kỳ domain nào trong reference, không có bằng chứng cụ thể.

**GraphRAG** (điểm: Comprehensiveness=3, Faithfulness=5, Multi-hop=4):
> Trả lời đúng 2/3 domain có bằng chứng cụ thể: "Smart Home Technology" (Aqara + FP2 Presence Sensor), "Display Technology" (Sensor OLED Display + organic photodiode) — trích đúng chunk_id làm bằng chứng.

### Root-cause analysis
- **Vì sao Flat RAG thất bại:** top-k=6 chunk lấy theo similarity ngữ nghĩa với câu hỏi chung ("Samsung", "technology domains") có thể trả về các chunk mô tả Samsung ở mức tổng quan (business overview) thay vì các chunk cụ thể nói về từng sản phẩm/quan hệ (Aqara, Sensor OLED Display) — vector search không ưu tiên được các cụm thông tin rời rạc nhưng liên quan tới nhiều thực thể khác nhau của cùng 1 công ty.
- **Vì sao GraphRAG giải quyết được:** đồ thị có sẵn các cạnh trực tiếp nối Samsung với từng công nghệ cụ thể (qua quan hệ DEVELOPED/PARTNERED_WITH/USES đã trích xuất), nên BFS 2-hop từ seed "Samsung" thu thập đúng các node liên quan mà vector search bỏ lỡ.
- **Điểm chưa hoàn hảo của GraphRAG:** domain thứ 3 ("Organic Photodiode Technology") bị Judge nhận xét là trùng lặp/thừa với domain thứ 2 — thiếu domain "Semiconductors" trong reference. Nguyên nhân nhiều khả năng là NER+RE không trích được quan hệ Samsung–Semiconductor trong tập 400 chunk giới hạn của lab.

---

## Ca lỗi 2 — GraphRAG thất bại (Flat RAG tốt hơn)

- **Question ID:** `G5000-29` (cross-doc)
- **Câu hỏi:** "How did participation in White House AI commitments broaden from July to September 2023 according to the selected reports?"
- **Reference:** Tháng 7 có 7 công ty (gồm Google, Meta, OpenAI) cam kết tự nguyện; tháng 9 mở rộng thêm IBM, Adobe, Salesforce và 5 công ty khác.

**Flat RAG** (điểm: Comprehensiveness=5, Faithfulness=5, Multi-hop=5):
> Trả lời đầy đủ và chính xác cả 2 mốc thời gian, đúng tên 3 công ty đại diện mỗi đợt.

**GraphRAG** (điểm: Comprehensiveness=3, Faithfulness=4, Multi-hop=3):
> Sai lệch: khẳng định "trong tháng 7, chỉ có OpenAI cam kết" — bỏ sót Google và Meta dù đề bài/reference đều nêu rõ có 7 công ty bao gồm cả 3 cái tên này.

### Root-cause analysis
- **Nguyên nhân gốc rễ:** đây là dạng câu hỏi `cross-doc` cần **liệt kê đầy đủ tập thực thể** tham gia 1 sự kiện (COMMITTED_TO) tại 1 thời điểm — nhưng NER+RE extraction (giới hạn 400 chunk, ưu tiên precision) nhiều khả năng chỉ trích được quan hệ `OpenAI -COMMITTED_TO-> White House` mà bỏ sót Google/Meta trong cùng bài báo, khiến đồ thị chỉ có 1 cạnh thay vì 3. GraphRAG bị giới hạn bởi **độ đầy đủ của đồ thị đã trích xuất**, trong khi Flat RAG đọc trực tiếp nguyên văn chunk gốc nên không mất thông tin.
- **Bằng chứng:** context đồ thị (`graph_debug`) chỉ chứa cạnh liên quan tới seed entity khớp được qua `match_seeds()` — nếu Google/Meta không có node/cạnh COMMITTED_TO tương ứng trong Neo4j (do bị bỏ sót ở bước extraction), GraphRAG không thể "phát minh" lại thông tin đó dù đã đi đúng hướng traversal.
- **Đề xuất khắc phục:** (1) tăng `EXTRACTION_MAX_CHUNKS` hoặc giảm ngưỡng "precision over recall" trong `EXTRACT_SYSTEM` prompt để bắt được nhiều quan hệ đồng xuất hiện hơn trong 1 chunk; (2) với câu hỏi dạng liệt kê tập hợp, kết hợp thêm context Vector (đã có sẵn trong Hybrid prompt `=== VECTOR ===`) làm nguồn bổ sung khi đồ thị thiếu cạnh — cần kiểm tra tại sao phần Vector context trong prompt hybrid không giúp model bù lại thông tin bị thiếu ở nhánh Graph.

---

## Tổng kết pattern lỗi

Cả 2 ca cho thấy **GraphRAG mạnh khi câu hỏi cần nối các thực thể rời rạc bằng quan hệ rõ ràng (Ca 1)**, nhưng **yếu khi câu hỏi cần liệt kê đầy đủ một tập hợp thực thể tham gia cùng 1 sự kiện (Ca 2)** — vì chất lượng GraphRAG bị chặn trên bởi độ đầy đủ (recall) của bước NER+RE extraction, một bước vốn được thiết kế ưu tiên precision trong giới hạn Scale Guard của lab (`EXTRACTION_MAX_CHUNKS=400`). Đây là trade-off có chủ đích của lab, không phải lỗi thiết kế ngẫu nhiên.
