# Reflection — Ngô Hữu Nghĩa

## 1. Mapping bài giảng vào implementation

| Khái niệm | Implementation thực tế | Quan sát và đánh giá |
|---|---|---|
| Conservative coreference | `resolve_coref_batch()`, `run_coref()`, `resolve_coref_single()` | Prompt chỉ cho phép resolve trong cùng chunk; fallback giữ nguyên text khi batch lỗi. Run có 60 chunk cần retry, nên cần theo dõi retry đến trạng thái hoàn tất thay vì coi fallback là thành công. |
| Schema và provenance guard | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `validate_before_insert()` | Edge được bắt buộc `source_chunk_id`, `published_date`, `evidence`, `confidence`; post-check cho 0 edge thiếu provenance. Đây là deterministic gate trước/sau ingest, không giao cho LLM tự quyết. |
| Entity resolution | `build_resolution_map()`, `merge_guard()`, `UF`, `canonicalize_triples()` | ANN sinh candidate; threshold 0.90 và lexical guard 0.72 quyết định merge; Union-Find tạo canonical ID. Audit run hiện có 1 merge `X90A`/`X90`, cho thấy cần bổ sung test negative và audit coverage. |
| Bulk graph ingestion | `build_nodes()`, `bulk_insert_nodes()`, `bulk_insert_edges()` | Neo4j nhận batch `UNWIND $rows AS row`, tránh round-trip từng record. Kết quả 223 node, 136 edge. |
| Hybrid retrieval và super-node cap | `retrieve_flat_context()`, `retrieve_graph_context()`, `textualize()` | Graph tối đa 2 hop, cap node lớn 50 edge và cap toàn cục 250 edge/14,000 ký tự; sample chưa có node >100, do đó policy cần được test bằng fixture hoặc dữ liệu lớn hơn. |
| Evaluation | `run_evaluation()`, `judge_answer()`, `comparison_table()` | Cả Flat/Graph chạy cùng 50 câu và cùng reference answer; output tách metric judge với latency/token đo được. Không suy luận “tốt hơn” nếu không chỉ rõ metric/dataset. |

## 2. Debugging và bài học

Vấn đề phức tạp nhất là lỗi coreference theo batch: notebook báo `Chunks cần retry: 60`, còn output retry mới ở `0/60`. Nếu tiếp tục extraction như thể các chunk đã được phân giải, pipeline có thể che giấu trạng thái chưa hoàn thành. Cách xử lý đúng là giữ nguyên văn bản, gắn `COREF_BATCH_FAILED`, checkpoint từng chunk và chỉ chuyển chunk sang extraction khi retry thành công hoặc được đánh dấu unresolved có chủ đích.

Bài học là LLM nên đề xuất cấu trúc, còn code phải áp gate xác định được: validate JSON/schema, provenance, threshold/guard, global edge cap và checkpoint. Ví dụ G5000-48 cho thấy context graph thiếu một record đã làm GraphRAG kém Flat RAG; “có graph” không thay thế được kiểm tra coverage theo từng ý hỏi.

## 3. Action plan áp dụng thực tế

**Đề án dự kiến:** trợ lý hỏi đáp tri thức kỹ thuật nội bộ cho tài liệu kiến trúc, runbook, incident và quyết định kỹ thuật.

Không nên dùng GraphRAG cho mọi câu hỏi. Flat RAG đủ cho tra cứu một tài liệu hoặc một fact đơn lẻ. Hybrid GraphRAG chỉ bật khi query cần nối service–incident–owner–runbook qua nhiều tài liệu, hoặc cần kiểm tra tác động/quan hệ phụ thuộc.

Schema ban đầu, có versioning và allowlist:

- Nodes: `Service`, `System`, `Document`, `Incident`, `Runbook`, `Team`, `Owner`, `Technology`, `Decision`.
- Relations: `DEPENDS_ON`, `OWNED_BY`, `DOCUMENTED_IN`, `AFFECTED_BY`, `MITIGATED_BY`, `SUPERSEDES`, `USES`.
- Mỗi edge phải có `source_document_id`, revision/timestamp, evidence span và confidence; không trả lời khẳng định nếu không còn evidence sau retrieval.

Entity resolution sẽ ưu tiên ID ổn định từ CMDB/repository trước, sau đó mới dùng alias + ANN candidate. Các merge có score sát ngưỡng hoặc khác loại entity phải vào hàng đợi human approval. Với super-node như shared platform/team, áp cap theo truy vấn và thời gian; đồng thời giữ diversity theo relation/type để không chỉ lấy các edge mới nhất. Retrieval phải log `request_id`, latency, token, model/provider, seeds, edge count, provenance coverage, guardrail decision và lỗi; tuyệt đối không log secrets.

## 4. Tự đánh giá

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Hiểu kiến trúc GraphRAG | 4 | Nắm được pipeline và các failure mode; cần thêm thử nghiệm dữ liệu lớn để kiểm chứng super-node. |
| Kiểm soát AI Coding Agent | 4 | Ưu tiên deterministic gate và không chấp nhận `O(N²)` toàn cục; cần hoàn thiện test negative cho guard. |
| Chất lượng knowledge graph | 3 | Provenance đạt, nhưng graph còn nhỏ và audit entity chỉ một dòng. |
| Phân tích/debug hệ thống | 4 | Đã xác định coreference retry và retrieval coverage là rủi ro cần đóng trước khi production. |
