# Thuyết minh kỹ thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Ngô Hữu Nghĩa  
**Phạm vi bằng chứng:** notebook đã chạy, `outputs/graphrag_eval_results.csv` (50 câu) và `outputs/graphrag_vs_flatrag_summary.csv`.

## 1. Coreference Resolution

Pipeline dùng prompt *conservative*: chỉ thay đại từ khi antecedent xuất hiện rõ trong cùng chunk; nếu không chắc chắn thì giữ nguyên văn bản và log `unresolved_mentions`. Đây là lựa chọn fail-closed vì một coreference sai sẽ biến một quan hệ đúng trong câu thành false edge trong graph.

Một tình huống khó thực tế là `chunk_id=8dc0de1ac5eca8c4bb72::c0000`: đoạn văn có “The company’s multiple products…” rồi mô tả Akeneo/product-information services, nhưng phần antecedent không đủ rõ để gán “The company” chắc chắn. Quan trọng hơn, run batch đã đánh dấu **60 chunk** là `COREF_BATCH_FAILED`; trong đó có chính chunk này. Cơ chế fallback giữ `text` gốc thay vì tự thay thế. Retry từng chunk được khởi tạo nhưng output notebook mới thể hiện tiến trình `0/60`, vì vậy không thể khẳng định 60 chunk đã được phân giải thành công.

Nếu ép gán “The company” thành Akeneo khi ngữ cảnh thực ra đang nói tới một tổ chức khác, RE có thể tạo edge sai, ví dụ `Akeneo -DEVELOPED/USES-> ...`. Edge sai có provenance vẫn truy vết được, nhưng sẽ làm BFS lấy nhầm ngữ cảnh và gây câu trả lời có vẻ có dẫn chứng nhưng sai về chủ thể.

## 2. Entity Resolution: ngưỡng và lexical guard

- Ngưỡng vector trong `build_resolution_map()` là **cosine similarity ≥ 0.90** với `sentence-transformers/all-MiniLM-L6-v2` và FAISS inner-product trên embedding đã chuẩn hoá.
- Lexical guard so sánh tên sau khi bỏ suffix doanh nghiệp (`Inc`, `Corp`, `LLC`, …); chỉ merge nếu hai chuỗi bằng nhau hoặc `SequenceMatcher` ≥ **0.72**. Resolution tách theo `entity_type`, rồi dùng Union-Find để tạo canonical entity.
- Bằng chứng audit thực tế chỉ có **một** dòng: `Technology: X90A` vs `X90`, similarity `0.920579`, quyết định `MERGE_VECTOR`. Đây là một merge hợp lệ về mặt guard, không phải reject.

Không có cặp similarity cao bị `REJECT_GUARD` trong output notebook hiện tại, nên không thể trích dẫn trung thực một ví dụ “>0.85 bị chặn” như yêu cầu đề bài. Đây là thiếu hụt coverage của audit, không phải bằng chứng guard đã xử lý mọi false merge. Test bổ sung cần đưa các negative pair có kiểm soát như `Apple`/`Apple Watch` và `Sam Altman`/`Steve Altman`, xác nhận chúng tạo `REJECT_GUARD` trước khi cho phép mở rộng pipeline.

## 3. Đồ thị và chính sách super-node

Run thực tế có **223 node**, **136 edge**, và `invalid_provenance_edges = 0`. Top degree từ `graph_checks()` là:

| Hạng | Entity | Type | Degree |
|---:|---|---|---:|
| 1 | ServiceNow | Company | 6 |
| 2 | Intelligent Technical Solutions | Company | 3 |
| 3 | Opsys Tech | Company | 3 |

Không node nào đạt ngưỡng `SUPER_NODE_DEGREE = 100`; vì vậy cap 50 cạnh chưa được kích hoạt trong dữ liệu lab này. Chính sách khi scale là: với node degree >100, lấy tối đa 50 cạnh theo `published_date` giảm dần, toàn truy vấn bị chặn thêm bởi `GLOBAL_EDGE_CAP=250` và `MAX_GRAPH_CONTEXT_CHARS=14000`.

Lợi ích là kiểm soát fan-out, latency và token/context explosion; ưu tiên bản tin mới phù hợp câu hỏi tình trạng hiện tại. Rủi ro là câu hỏi lịch sử hoặc quan hệ cũ có thể bị loại. Với truy vấn có tín hiệu thời gian quá khứ, retrieval nên chuyển sang lọc theo khoảng ngày hoặc bổ sung một quota cạnh theo thời kỳ thay vì chỉ sort mới nhất.

## 4. Benchmark Flat RAG và GraphRAG

Toàn bộ 50 câu được chạy trên cùng golden dataset. Các điểm chất lượng là **LLM-as-a-Judge**, vì vậy dùng để so sánh định hướng chứ không thay thế metric xác định tuyệt đối.

| Metric trung bình | Flat RAG | GraphRAG | Δ Graph−Flat |
|---|---:|---:|---:|
| Comprehensiveness (1–5) | 3.04 | 3.24 | +0.20 |
| Faithfulness (1–5) | 4.84 | 4.92 | +0.08 |
| Multi-hop reasoning (1–5) | 3.36 | 3.74 | +0.38 |
| Latency (s) | 7.41 | 7.17 | −0.24 |
| Token usage | 1667.14 | 1676.34 | +9.20 |

Theo nhóm, GraphRAG nổi bật nhất ở `cross-doc`: comprehensive 3.455 vs 3.136 và multi-hop 3.682 vs 3.364. Ở `multi-hop`, comprehensiveness bằng nhau (3.174) nhưng graph tăng multi-hop reasoning 3.913 vs 3.522; đổi lại latency 8.367s vs 7.414s và token 1776.652 vs 1725.696.

**Ca Flat RAG thất bại, GraphRAG thành công — G5000-45.** Câu hỏi yêu cầu tránh đếm đôi hai bản tin về LTTS/Qualcomm được Thales lựa chọn. Flat RAG chỉ từ chối do context thiếu Qualcomm (comprehensiveness/multi-hop: 1/1). GraphRAG đề xuất canonicalize thành một contract/event node, gắn các bên tham gia và chỉ aggregate contract một lần; Judge đánh giá đây là hướng phù hợp với reference answer (comprehensiveness tăng 1→4, multi-hop 1→5). Tuy vậy, answer GraphRAG cũng thừa nhận chưa có evidence Qualcomm; phần mô hình contract là suy luận thiết kế, cần gắn cờ inference và cần source bổ sung trước khi khẳng định dữ kiện.

**Ca GraphRAG khó hơn Flat RAG — G5000-48.** Câu hỏi yêu cầu ghép ba record Snowflake: H2O AI Cloud, CRN Big Data 100, và tín hiệu nhu cầu. Flat RAG trả đủ ba ý (5/5/5). GraphRAG bỏ record CRN Big Data 100, thay bằng nhận định “Data Cloud leader” không có trong context; kết quả 3/2/2. Nguyên nhân hợp lý là graph/vector context không thu đủ ba chứng cứ và linearization ưu tiên cạnh khác. Cần thêm coverage gate: trước khi sinh answer, kiểm tra mỗi ý trong truy vấn có ít nhất một provenance; nếu thiếu thì mở rộng retrieval hoặc trả lời thiếu bằng chứng.

## 5. Trade-off, kiểm soát Coding Agent và scale

Flat RAG đơn giản hơn: chỉ embedding + top-k chunk, index/ingestion rẻ và ít failure mode. GraphRAG có thêm chi phí extraction, canonicalization, Neo4j ingestion, traversal và quản trị provenance, nhưng đem lại cấu trúc để nối thực thể/sự kiện qua nhiều tài liệu. Sample hiện tại cho thấy quality tăng vừa phải và token gần tương đương; không đủ bằng chứng để tuyên bố GraphRAG luôn tốt hơn.

Một đề xuất không nên chấp nhận là pairwise cosine toàn cục `O(N²)` cho entity/near-duplicate trên toàn bộ 350MB: tốn RAM/thời gian, khó audit và dễ OOM. Thiết kế hiện tại dùng ANN/FAISS để sinh candidate, lexical guard để quyết định merge, và Union-Find để gom cụm. Khi scale, bottleneck đầu tiên là LLM coreference + triple extraction (chi phí, rate limit, lỗi batch), không phải BFS. Hướng xử lý: queue/batch bất đồng bộ có retry-idempotency; checkpoint theo chunk; validate schema/provenance trước ingest; ANN blocking theo loại entity; `UNWIND` bulk batch; và quan sát structured logs không chứa secret. Merge mơ hồ phải vào audit/HITL, không tự động hoá bằng LLM.

## Kết luận

Pipeline đã chứng minh được provenance đầy đủ (0 edge thiếu trường bắt buộc), retrieval hybrid và benchmark 50 câu. Các giới hạn cần được ghi nhận rõ: coreference retry chưa có bằng chứng hoàn tất, entity audit chưa có negative/reject pair, và test super-node chưa được kích hoạt vì degree cao nhất chỉ là 6.
