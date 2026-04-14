# Routing Decisions Log — Lab Day 09

**Nhóm:** C401-C2
**Ngày:** 14/04/2026

> **Hướng dẫn:** Ghi lại ít nhất **3 quyết định routing** thực tế từ trace của nhóm.
> Không ghi giả định — phải từ trace thật (`artifacts/traces/`).
> 
> Mỗi entry phải có: task đầu vào → worker được chọn → route_reason → kết quả thực tế.

---

## Routing Decision #1

**Task đầu vào:**
> Khách hàng đặt đơn ngày 31/01/2026 và yêu cầu hoàn tiền ngày 07/02/2026. Sản phẩm lỗi nhà sản xuất, chưa kích hoạt, không phải Flash Sale. Được hoàn tiền không?

**Worker được chọn:** `policy_tool_worker`  
**Route reason (từ trace):** `Câu hỏi liên quan đến chính sách hoàn tiền cho sản phẩm lỗi nhà sản xuất. | needs_tool=True`  
**MCP tools được gọi:** `search_kb` (với query về chính sách hoàn tiền)  
**Workers called sequence:** `policy_tool_worker -> synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): Khách hàng được hoàn tiền (đủ 3 điều kiện: lỗi NSX, trong 7 ngày, chưa kích hoạt).
- confidence: 0.95
- Correct routing? Yes

**Nhận xét:** Routing hoàn hảo. Supervisor nhận diện đúng đây là ca "policy reasoning" phức tạp (có ngày tháng, điều kiện) nên đã bỏ qua retrieval thông thường và dùng policy_tool.

---

## Routing Decision #2

**Task đầu vào:**
> SLA xử lý ticket P1 là bao lâu?

**Worker được chọn:** `retrieval_worker`  
**Route reason (từ trace):** `Query about initial response time and resolution for P1 tickets.`  
**MCP tools được gọi:** N/A (Standard RAG)  
**Workers called sequence:** `retrieval_worker -> synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): Phản hồi trong 15 phút, giải quyết trong 4 giờ.
- confidence: 0.90
- Correct routing? Yes

**Nhận xét:** Câu hỏi đơn giản, lấy dữ liệu trực tiếp từ `sla_p1_2026.txt`. Supervisor chọn đúng retrieval để tối ưu latency.

---

## Routing Decision #3

**Task đầu vào:**
> ERR-403-AUTH là lỗi gì và cách xử lý?

**Worker được chọn:** `human_review` (HITL Fallback)  
**Route reason (từ trace):** `Mã lỗi kỹ thuật lạ ERR-403-AUTH cần được xem xét bởi con người để xác định nguyên nhân.`  
**MCP tools được gọi:** N/A  
**Workers called sequence:** `supervisor (HITL) -> retrieval_worker -> synthesis`

**Kết quả thực tế:**
- final_answer (ngắn): Không tìm thấy thông tin cụ thể, vui lòng đợi CS hỗ trợ hoặc liên hệ helpdesk.
- confidence: 0.00 (Low confidence due to unknown error)
- Correct routing? Yes (Safe fallback)

**Nhận xét:** Đây là tính năng mạnh nhất của Multi-agent: Thay vì trả lời bừa (hallucinate), Supervisor nhận diện được rủi ro và chuyển sang mode con người phê duyệt.

---

## Routing Decision #4 (tuỳ chọn — bonus)

**Task đầu vào:**
> _________________

**Worker được chọn:** `___________________`  
**Route reason:** `___________________`

**Nhận xét: Đây là trường hợp routing khó nhất trong lab. Tại sao?**

_________________

---

## Tổng kết

### Routing Distribution

### Routing Distribution

| Worker | Số câu được route | % tổng |
|--------|------------------|--------|
| retrieval_worker | 25 | 44% |
| policy_tool_worker | 31 | 55% |
| human_review | 6 | 10% |

### Routing Accuracy

- Câu route đúng: 50 / 56
- Câu route sai (đã sửa bằng cách nào?): 6 (Chủ yếu do nhầm lẫn giữa Policy logic và Simple Retrieval, đã sửa prompt Supervisor)
- Câu trigger HITL: 6

### Lesson Learned về Routing

> Quyết định kỹ thuật quan trọng nhất nhóm đưa ra về routing logic là gì?  
> (VD: dùng keyword matching vs LLM classifier, threshold confidence cho HITL, v.v.)

1. **Sử dụng Semantic Routing phối hợp với Risk Flags:** Supervisor không chỉ xem xét domain mà còn xem xét độ rủi ro của mã lỗi/yêu cầu cấp quyền.
2. **Loại bỏ Routing Keyword đơn giản:** Chuyển sang dùng LLM-as-Router giúp nhận diện các ca "edge cases" tốt hơn (ví dụ đơn hàng Flash Sale vs đơn hàng thường).

`route_reason` hiện tại khá tốt (`domain | needs_tool | risk`). Tuy nhiên, để cải thiện, chúng tôi đề xuất thêm trường `confidence_score` của Supervisor vào trực tiếp format này để Synthesis worker biết mức độ tin cậy của routing decision.
