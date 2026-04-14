# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Bùi Minh Ngọc  
**Vai trò trong nhóm:** Trace & Docs Owner  
**Ngày nộp:** 2026-04-14  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- File chính: `graph.py`, `docs/routing_decisions.md`, `docs/single_vs_multi_comparison.md`
- Functions tôi implement: `supervisor_node()`, `route_decision()`, `make_initial_state()`, `save_trace()`

**Cách công việc của tôi kết nối với phần của thành viên khác:**

Tôi implement lớp điều phối trung tâm (`supervisor_node`) — đây là điểm khởi đầu của toàn bộ pipeline. Kết quả `supervisor_route` mà hàm này trả về quyết định worker nào được gọi tiếp theo (`retrieval_worker`, `policy_tool_worker` hoặc `human_review`). Nếu routing sai, toàn bộ downstream (retrieval, synthesis) sẽ nhận input không phù hợp và cho kết quả sai. Tôi cũng viết `routing_decisions.md` tổng hợp các trace thực tế và `single_vs_multi_comparison.md` so sánh định lượng giữa Day 08 và Day 09 để cả nhóm có cơ sở đánh giá kiến trúc.

**Bằng chứng:**

`graph.py` — phần Supervisor (line 97–162); `docs/routing_decisions.md`; `docs/single_vs_multi_comparison.md`

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Dùng **LLM-as-Router** (ChatOpenAI gpt-4o-mini) thay vì keyword matching để phân loại task trong `supervisor_node`.

**Lý do:**

Ban đầu tôi thử keyword matching đơn giản: nếu task chứa từ "hoàn tiền", "chính sách" thì route sang `policy_tool_worker`; ngược lại route sang `retrieval_worker`. Cách này nhanh (~5ms) nhưng thất bại với các câu hỏi edge-case — ví dụ câu hỏi "Khách hàng Flash Sale yêu cầu hoàn tiền vì sản phẩm lỗi nhà sản xuất" chứa cả hai domain và bị route nhầm. Tôi chuyển sang dùng LLM với system prompt mô tả rõ luật điều phối 3 worker, trả về JSON `{"route": ..., "reason": ..., "risk_high": bool, "needs_tool": bool}`. Cách này chính xác hơn đáng kể, đặc biệt với câu hỏi đa điều kiện.

**Trade-off đã chấp nhận:**

Latency tăng thêm ~800ms mỗi request (1 LLM call thêm ở Supervisor). Tuy nhiên, routing accuracy tăng từ ~70% lên ~89% (50/56 câu đúng theo `routing_decisions.md`), đánh đổi chấp nhận được.

**Bằng chứng từ trace/code:**

```python
# graph.py — supervisor_node(), line 108–137
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
system_prompt = (
    "Luật điều phối:\n"
    "1. 'policy_tool_worker': câu hỏi về quy định, chính sách cụ thể...\n"
    "2. 'retrieval_worker': câu hỏi tra cứu thông tin chung, FAQ, SLA.\n"
    "3. 'human_review': mã lỗi kỹ thuật lạ (ERR-XXX) hoặc yêu cầu khẩn cấp mập mờ.\n"
    "Định dạng JSON: {\"route\": ..., \"reason\": ..., \"risk_high\": bool, \"needs_tool\": bool}"
)
# Routing Decision #1 (routing_decisions.md):
# Task: "Khách hàng Flash Sale yêu cầu hoàn tiền vì sản phẩm lỗi NSX..."
# → route = policy_tool_worker | confidence = 0.95 ✅
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** `supervisor_node` route sang `policy_tool_worker` nhưng `route_reason` không đề cập MCP → Synthesis worker không biết MCP đã được gọi.

**Symptom (pipeline làm gì sai?):**

Khi chạy trace cho câu "Khách hàng Flash Sale yêu cầu hoàn tiền", pipeline route đúng sang `policy_tool_worker` nhưng `route_reason` chỉ ghi `"Câu hỏi liên quan đến chính sách hoàn tiền"`. Synthesis worker không thấy tín hiệu `needs_tool=True` trong reason nên không trích dẫn kết quả MCP, dẫn đến `final_answer` thiếu bằng chứng từ knowledge base.

**Root cause:**

LLM đôi khi trả về `route_reason` không đề cập từ "tool" hay "mcp" dù `needs_tool=True`. Synthesis worker đọc `route_reason` để quyết định cách format câu trả lời.

**Cách sửa:**

Thêm post-processing trong `supervisor_node` (line 139–141 `graph.py`): nếu `route == "policy_tool_worker"` và `route_reason` không chứa "mcp" hoặc "tool" thì tự động append `"| needs_tool=True, will use MCP"`.

```python
# graph.py line 139–141 — fix đã implement
if route == "policy_tool_worker" and \
   "mcp" not in route_reason.lower() and \
   "tool" not in route_reason.lower():
    route_reason += " | needs_tool=True, will use MCP"
```

**Bằng chứng trước/sau:**

```
# TRƯỚC khi sửa:
route_reason = "Câu hỏi liên quan đến chính sách hoàn tiền cho sản phẩm lỗi nhà sản xuất."
→ Synthesis bỏ qua MCP output, final_answer thiếu nguồn trích dẫn.

# SAU khi sửa:
route_reason = "Câu hỏi liên quan đến... | needs_tool=True, will use MCP"
→ confidence = 0.95, final_answer có đủ điều kiện hoàn tiền từ KB.
```

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**

Phần routing logic và documentation. `supervisor_node` hoạt động ổn định, xử lý đúng 3 loại câu hỏi với routing accuracy 89% (50/56). `single_vs_multi_comparison.md` có số liệu cụ thể từ trace thực tế (faithfulness +0.40, completeness +0.80, abstain rate giảm từ 30% xuống 10%) thay vì ước đoán.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Chưa xử lý tốt trường hợp LLM trả về JSON không hợp lệ — hiện chỉ fallback về `retrieval_worker` và ghi log lỗi, nhưng chưa có retry logic. 6 câu route sai đều liên quan đến edge-case giữa policy và retrieval.

**Nhóm phụ thuộc vào tôi ở đâu?**

Toàn bộ pipeline block ở `supervisor_node` — nếu routing logic chưa xong thì không thể test retrieval_worker, policy_tool_worker hay synthesis.

**Phần tôi phụ thuộc vào thành viên khác:**

Cần `workers/retrieval.py`, `workers/policy_tool.py`, và `workers/synthesis.py` từ các thành viên khác để `graph.py` chạy được end-to-end.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ thêm **`confidence_score` của Supervisor vào `route_reason`** vì trace của câu `gq09` (ERR-403-AUTH) cho thấy Supervisor route đúng sang `human_review` nhưng không có chỉ số nào cho Synthesis worker biết mức độ chắc chắn của quyết định đó (confidence luôn là `0.00` với HITL). Nếu Supervisor expose confidence (ví dụ: `"route_reason": "ERR code lạ | confidence=0.35 | HITL triggered"`), Synthesis có thể format câu trả lời như "Hệ thống chưa chắc chắn, vui lòng chờ CS xác nhận" thay vì trả về string rỗng.

---

*File: `reports/individual/BuiMinhNgoc_2A202600354.md`*
