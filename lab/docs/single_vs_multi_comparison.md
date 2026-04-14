# Single Agent vs Multi-Agent Comparison — Lab Day 09

**Nhóm:** C401-C2
**Ngày:** 14/04/2026

> **Hướng dẫn:** So sánh Day 08 (single-agent RAG) với Day 09 (supervisor-worker).
> Phải có **số liệu thực tế** từ trace — không ghi ước đoán.
> Chạy cùng test questions cho cả hai nếu có thể.

---

## 1. Metrics Comparison

> Điền vào bảng sau. Lấy số liệu từ:
> - Day 08: chạy `python eval.py` từ Day 08 lab
> - Day 09: chạy `python eval_trace.py` từ lab này

| Metric | Day 08 (Single Agent) | Day 09 (Multi-Agent) | Delta | Ghi chú |
|--------|----------------------|---------------------|-------|---------|
| Avg faithfulness | 3.80 | 4.20 | +0.40 | Cải thiện nhờ bước Re-rank/Synthesis riêng |
| Avg relevance | 3.80 | 4.15 | +0.35 | Supervisor lọc đúng domain worker |
| Avg context recall | 5.00 | 5.00 | 0.00 | Cả hai đều retrieve tốt |
| Avg completeness | 3.50 | 4.30 | +0.80 | Synthesis worker check đủ ý |
| Abstain rate (%) | 30% | 10% | -20% | Giảm nhờ HITL/MCP bổ sung dữ liệu |
| Avg latency (ms) | ~2000 ms | 15833 ms | +13833 ms | Multi-agent chậm hơn do nhiều bước LLM |
| Routing visibility | ✗ Không có | ✓ Có route_reason | N/A | Dễ dàng biết tại sao đi vào worker đó |
| Debug time (estimate) | 20-30 phút | 5-10 phút | -15 phút | Trace giúp khoanh vùng lỗi nhanh |

> **Lưu ý:** Nếu không có Day 08 kết quả thực tế, ghi "N/A" và giải thích.

---

## 2. Phân tích theo loại câu hỏi

### 2.1 Câu hỏi đơn giản (single-document)

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Accuracy | Cao (90%+) | Cao (95%+) |
| Latency | Thấp (1-2s) | Cao (10-15s) |
| Observation | Chạy thẳng retrieve -> gen. | Supervisor mất thêm 1 bước phân loại. |

**Kết luận:** Multi-agent không mang lại lợi thế lớn về độ chính xác ở level này, nhưng làm tăng latency đáng kể. Tuy nhiên, khả năng giải thích (explainability) tốt hơn.

_________________

### 2.2 Câu hỏi multi-hop (cross-document)

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Accuracy | Trung bình (50%) | Cao (85%+) |
| Routing visible? | ✗ | ✓ |
| Observation | Dễ lẫn lộn context. | Supervisor tách task thành các sub-tasks cho workers. |

**Kết luận:** Multi-agent thắng tuyệt đối ở các câu hỏi phức tạp nhờ sự chuyên môn hóa của các Worker (Policy vs Retrieval).

_________________

### 2.3 Câu hỏi cần abstain

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Abstain rate | 30% | 10% |
| Hallucination cases | Có (khi cố trả lời q09) | Ít (chuyển sang HITL) |
| Observation | Hay trả lời bừa hoặc báo lỗi. | HITL giúp xử lý các ca "unseen error codes". |

**Kết luận:** Multi-agent an toàn hơn nhờ cơ chế "Human-in-the-loop" và fallback logic trong Supervisor.

_________________

---

## 3. Debuggability Analysis

> Khi pipeline trả lời sai, mất bao lâu để tìm ra nguyên nhân?

### Day 08 — Debug workflow
```
Khi answer sai → phải đọc toàn bộ RAG pipeline code → tìm lỗi ở indexing/retrieval/generation
Không có trace → không biết bắt đầu từ đâu
Thời gian ước tính: 20-30 phút
```

### Day 09 — Debug workflow
```
Khi answer sai → đọc trace → xem supervisor_route + route_reason
  → Nếu route sai → sửa supervisor routing logic
  → Nếu retrieval sai → test retrieval_worker độc lập
  → Nếu synthesis sai → test synthesis_worker độc lập
Thời gian ước tính: 5-10 phút
```

**Câu cụ thể nhóm đã debug:** Câu hỏi `q09` (Mã lỗi ERR-403-AUTH). 
- **Day 08:** Answer báo "Không đủ dữ liệu" nhưng không rõ tại sao không tìm thấy.
- **Day 09:** Trace chỉ rõ Supervisor nhận diện đây là mã lỗi lạ và chủ động route sang `human_review` (HITL), giúp biết ngay lỗi không nằm ở Retrieval mà do thiếu document.

_________________

---

## 4. Extensibility Analysis

> Dễ extend thêm capability không?

| Scenario | Day 08 | Day 09 |
|---------|--------|--------|
| Thêm 1 tool/API mới | Phải sửa toàn prompt | Thêm MCP tool + route rule |
| Thêm 1 domain mới | Phải retrain/re-prompt | Thêm 1 worker mới |
| Thay đổi retrieval strategy | Sửa trực tiếp trong pipeline | Sửa retrieval_worker độc lập |
| A/B test một phần | Khó — phải clone toàn pipeline | Dễ — swap worker |

**Nhận xét:** Multi-agent là kiến trúc "cần thiết" nếu hệ thống có ý định mở rộng sang nhiều domain hoặc tích hợp nhiều tool (Jira, Slack, SQL). Day 08 bị giới hạn bởi context window và prompt complexity.

_________________

---

## 5. Cost & Latency Trade-off

> Multi-agent thường tốn nhiều LLM calls hơn. Nhóm đo được gì?

| Scenario | Day 08 calls | Day 09 calls |
|---------|-------------|-------------|
| Simple query | 1 LLM call | 2 LLM calls (Supervisor + Synthesis) |
| Complex query | 1 LLM call | 3+ LLM calls (Sup + Worker + Synthesis) |
| MCP tool call | N/A | 0-1 LLM call |

**Nhận xét về cost-benefit:** multi-agent đắt hơn về token và latency nhưng "đáng tiền" ở khả năng kiểm soát flow và độ tin cậy.

_________________

---

## 6. Kết luận

1. **Khả năng quan sát (Observability):** Trace file cung cấp cái nhìn chi tiết vào quá trình suy luận.
2. **Khả năng mở rộng (Extensibility):** Dễ dàng cắm thêm Worker hoặc Tool qua MCP mà không phá vỡ logic cũ.

> **Multi-agent kém hơn hoặc không khác biệt ở điểm nào?**

1. **Latency:** Chậm hơn rõ rệt (gấp 8-10 lần) do phải qua nhiều bước LLM nối tiếp.

> **Khi nào KHÔNG nên dùng multi-agent?**

Khi bài toán đơn giản (chỉ cần search 1 tập doc duy nhất) và yêu cầu phản hồi nhanh (real-time chat).

> **Nếu tiếp tục phát triển hệ thống này, nhóm sẽ thêm gì?**

Thêm bộ nhớ hội thoại (Session management) giữa các worker và cơ chế Retry tự động nếu Worker trả về output không hợp lệ.
