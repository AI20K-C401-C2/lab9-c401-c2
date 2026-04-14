# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Phan Tuấn Minh  
**Vai trò trong nhóm:** Worker Owner  
**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

> **Lưu ý quan trọng:**
> - Viết ở ngôi **"tôi"**, gắn với chi tiết thật của phần bạn làm
> - Phải có **bằng chứng cụ thể**: tên file, đoạn code, kết quả trace, hoặc commit
> - Nội dung phân tích phải khác hoàn toàn với các thành viên trong nhóm
> - Deadline: Được commit **sau 18:00** (xem SCORING.md)
> - Lưu file với tên: `reports/individual/phan_tuan_minh.md`

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

> Mô tả cụ thể module, worker, contract, hoặc phần trace bạn trực tiếp làm.

**Module/file tôi chịu trách nhiệm:**
- File chính: `workers/policy_tool.py`
- Files phụ tôi đóng góp: `workers/synthesis.py`, `workers/retrieval.py`
- Functions tôi implement: `analyze_policy()`, `run()`, `_call_mcp_tool()` trong policy_tool.py; `_llm_judge_confidence()`, sửa `_estimate_confidence()` trong synthesis.py

Tôi là Worker Owner, chịu trách nhiệm implement logic cho 3 worker agents: Retrieval Worker, Policy Tool Worker, và Synthesis Worker. Mỗi worker đều tuân theo contract đã định nghĩa trong `contracts/worker_contracts.yaml`, đảm bảo input/output rõ ràng và test độc lập được.

**Cách công việc của tôi kết nối với phần của thành viên khác:**

Output `policy_result` từ `policy_tool.py` của tôi được `synthesis_worker` sử dụng để tổng hợp câu trả lời cuối cùng. Supervisor trong `graph.py` quyết định khi nào route task đến worker của tôi dựa vào nội dung câu hỏi. Retrieval Worker cung cấp `retrieved_chunks` làm context cho Policy Worker phân tích. Các worker của tôi cũng gọi MCP tools qua `mcp_server.py`.

**Bằng chứng:**

Code có comment trong `workers/policy_tool.py`, `workers/synthesis.py`. Contract status đã cập nhật `"done"` trong `worker_contracts.yaml` (dòng 99, 169, 217).

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi chọn dùng **rule-based exception detection** kết hợp **LLM fallback** trong hàm `analyze_policy()` thay vì chỉ dùng LLM thuần túy.

**Lý do:**

Rule-based approach có ưu điểm: nhanh (không cần gọi API), deterministic (cùng input luôn cho cùng output), và không phụ thuộc vào API key hay kết nối mạng. Tôi đã hardcode 3 exception cases quan trọng nhất: Flash Sale (`flash_sale_exception`), sản phẩm kỹ thuật số (`digital_product_exception`), và sản phẩm đã kích hoạt (`activated_exception`). Sau khi rule-based check xong, tôi thêm LLM call bổ sung để phân tích phức tạp hơn, nhưng bọc trong `try/except` để nếu LLM fail thì vẫn có kết quả từ rules.

**Trade-off đã chấp nhận:**

Rule-based chỉ detect được các exception đã biết trước. Nếu có exception mới (ví dụ chính sách v5 thêm rule mới), phải thêm code thủ công. LLM có thể phát hiện pattern mới nhưng chậm hơn (~800ms) và không deterministic. Tôi chấp nhận trade-off này vì trong lab, tập exception đã xác định rõ và ổn định.

**Bằng chứng từ trace/code:**

```python
# workers/policy_tool.py — dòng 86-108
# Exception 1: Flash Sale
if "flash sale" in task_lower or "flash sale" in context_text:
    exceptions_found.append({
        "type": "flash_sale_exception",
        "rule": "Đơn hàng Flash Sale không được hoàn tiền (Điều 3, chính sách v4).",
        "source": "policy_refund_v4.txt",
    })

# Exception 2: Digital product
if any(kw in task_lower for kw in ["license key", "license", "subscription", "kỹ thuật số"]):
    exceptions_found.append({
        "type": "digital_product_exception",
        "rule": "Sản phẩm kỹ thuật số (license key, subscription) không được hoàn tiền (Điều 3).",
        "source": "policy_refund_v4.txt",
    })
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Đoạn gọi LLM trong `policy_tool.py` ban đầu không có `try/except`, nếu API key thiếu hoặc mạng lỗi thì cả hàm `analyze_policy()` sẽ crash và pipeline bị dừng hoàn toàn.

**Symptom (pipeline làm gì sai?):**

Khi uncomment đoạn gọi OpenAI trong `analyze_policy()` mà không có `OPENAI_API_KEY` trong `.env`, hàm raise `AuthenticationError` và toàn bộ policy_tool_worker crash. State không được cập nhật `policy_result`, dẫn đến synthesis_worker nhận `policy_result = {}` và không thể phân tích exceptions.

**Root cause:**

Import `from openai import OpenAI` và gọi `client.chat.completions.create()` trực tiếp trong function body mà không có error handling. Khi API key thiếu hoặc không hợp lệ, exception propagate lên và không có fallback.

**Cách sửa:**

Bọc toàn bộ đoạn gọi LLM trong `try/except Exception`, nếu fail thì gán `analysis = "Rule-based analysis (LLM không khả dụng)."` làm fallback. Điều này đảm bảo hàm luôn trả về kết quả hợp lệ, rule-based results vẫn hoạt động bình thường.

**Bằng chứng trước/sau:**

```python
# TRƯỚC (crash nếu không có API key):
from openai import OpenAI
client = OpenAI()
response = client.chat.completions.create(...)
analysis = response.choices[0].message.content

# SAU (an toàn với fallback):
try:
    from openai import OpenAI
    client = OpenAI()
    response = client.chat.completions.create(...)
    analysis = response.choices[0].message.content
except Exception:
    analysis = "Rule-based analysis (LLM không khả dụng)."
```

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**

Tôi implement đúng contract cho cả 3 workers, đặc biệt policy_tool_worker xử lý chính xác các exception cases (Flash Sale, digital product, activated product). Mỗi worker đều có standalone test (`if __name__ == "__main__"`) để verify độc lập trước khi kết nối vào graph. Tôi cũng thêm LLM-as-Judge trong synthesis.py để cải thiện chất lượng confidence scoring.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Chưa hoàn thiện việc kết nối workers vào graph.py (3 hàm node vẫn dùng placeholder thay vì gọi worker thật). LLM-based policy analysis cũng chưa được test kỹ với nhiều edge cases.

**Nhóm phụ thuộc vào tôi ở đâu?**

Synthesis worker cần `policy_result` từ policy_tool_worker của tôi để tổng hợp câu trả lời. Nếu tôi chưa xong, pipeline không có thông tin exceptions để cảnh báo user.

**Phần tôi phụ thuộc vào thành viên khác:**

Tôi cần Supervisor Owner hoàn thiện routing logic trong graph.py để route đúng task đến policy_tool_worker. Tôi cũng cần MCP Owner implement `mcp_server.py` để các MCP tool calls hoạt động thật thay vì mock.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ hoàn thiện việc kết nối 3 workers vào graph.py (uncomment import và thay placeholder bằng `return xxx_run(state)`) rồi chạy end-to-end test với các câu hỏi thật. Ngoài ra, tôi sẽ nâng cấp `_llm_judge_confidence` trong synthesis.py để đánh giá thêm tiêu chí "câu trả lời có xử lý đúng exceptions không", vì hiện tại trace cho thấy confidence score chưa phản ánh chính xác chất lượng câu trả lời trong các case có nhiều exceptions.

---

*File: `reports/individual/phan_tuan_minh.md`*
