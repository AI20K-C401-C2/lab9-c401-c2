# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Phạm Việt Anh  
**Vai trò trong nhóm:** Supervisor Owner / Worker Owner / MCP Owner / Trace & Docs Owner  
**Ngày nộp:** 14/4/2026  
**Độ dài yêu cầu:** 500–800 từ

---

> **Lưu ý quan trọng:**
> - Viết ở ngôi **"tôi"**, gắn với chi tiết thật của phần bạn làm
> - Phải có **bằng chứng cụ thể**: tên file, đoạn code, kết quả trace, hoặc commit
> - Nội dung phân tích phải khác hoàn toàn với các thành viên trong nhóm
> - Deadline: Được commit **sau 18:00** (xem SCORING.md)
> - Lưu file với tên: `reports/individual/[ten_ban].md` (VD: `nguyen_van_a.md`)

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

Trong Lab 09 hôm nay, tôi đảm nhận vai trò **Supervisor Owner** (Người sở hữu bộ điều phối). Công việc chính của tôi là chuyển đổi hệ thống RAG đơn luồng từ Lab 08 sang kiến trúc **Multi-Agent** sử dụng framework **LangGraph**.

**Module/file tôi chịu trách nhiệm:**
- File chính: `graph.py`
- Functions tôi implement: `supervisor_node`, `build_graph`, `AgentState`, `route_decision`.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Tôi xây dựng khung sườn (orchestration layer) cho toàn bộ hệ thống. Công việc của tôi kết nối trực tiếp với các Worker: nhận kết quả từ `retrieval_worker` để chuyển tiếp cho `policy_tool_worker`, và cuối cùng thu thập dữ liệu từ các Agent để `synthesis_worker` tổng hợp câu trả lời. Tôi cũng thiết lập cơ chế rẽ nhánh để gọi `human_review` khi hệ thống gặp các trường hợp rủi ro cao.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**
Các thay đổi lớn trong file `graph.py` với cấu trúc `StateGraph` và logic điều hướng bằng LLM., commit:update sprint1 Việt Anh

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi quyết định sử dụng framework **LangGraph** để xây dựng kiến trúc **Supervisor-Worker** thay vì sử dụng luồng mã Python tuần tự (sequential control flow) truyền thống.

**Lý do:**
Việc sử dụng LangGraph mang lại hai lợi ích cốt lõi cho hệ thống RAG đa tác vụ này. Thứ nhất là khả năng **quản lý trạng thái tập trung (`AgentState`)**. Trong một hệ thống đa Agent, việc biết Agent nào đã làm gì và dữ liệu đang ở giai đoạn nào là cực kỳ quan trọng; LangGraph tự động hóa việc này, giúp tránh lỗi chồng chéo dữ liệu. Thứ hai là **tính linh hoạt trong điều hướng**. Thay vì dùng các câu lệnh `if/else` lồng nhau phức tạp để kiểm tra rủi ro hay loại câu hỏi, tôi sử dụng `conditional_edges` (cạnh rẽ nhánh có điều kiện). Điều này giúp Supervisor có thể chuyển đổi linh hoạt giữa việc tra cứu thông tin (Retrieval) hoặc kiểm tra chính sách (Policy) dựa trên đánh giá của LLM.

**Trade-off đã chấp nhận:**
Đổi lại, hệ thống có độ phức tạp ban đầu cao hơn. Tôi phải mất thêm thời gian để cấu hình các `node` và `edge` theo đúng chuẩn của framework, cũng như xử lý các lỗi tương thích về kiểu dữ liệu trong `StateGraph`. Tuy nhiên, sự đánh đổi này là xứng đáng để hệ thống có khả năng mở rộng (scalability) tốt hơn.

**Bằng chứng từ trace/code:**

```python
# File graph.py
workflow = StateGraph(AgentState)
workflow.add_node("supervisor", supervisor_node)
workflow.add_conditional_edges(
    "supervisor",
    route_decision,
    {
        "retrieval_worker": "retrieval_worker",
        "policy_tool_worker": "policy_tool_worker",
        "human_review": "human_review"
    }
)
```


---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** `UnicodeEncodeError` khi in kết quả tiếng Việt ra console trên Windows.

**Symptom (pipeline làm gì sai?):**
Khi chạy các câu truy vấn kiểm thử (test queries) có chứa tiếng Việt, chương trình bị crash ngay lập tức với thông báo lỗi: `UnicodeEncodeError: 'charmap' codec can't encode character...`. Điều này khiến tôi không thể kiểm tra được kết quả điều phối của Supervisor.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**
Nguyên nhân nằm ở sự không tương thích giữa bảng mã mặc định của Windows console (cp1252/cp1258) và các ký tự Unicode/Vietnamese trong chuỗi ký tự mà tôi dùng trong `print()`. Python cố gắng mã hóa các ký tự này sang bảng mã của console nhưng thất bại.

**Cách sửa:**
Tôi đã thực hiện hai bước. Thứ nhất là loại bỏ các ký tự Unicode đặc biệt như `▶` hay `⚠️` trong các câu lệnh in. Thứ hai, quan trọng nhất là thêm đoạn mã cấu hình lại `sys.stdout` ngay đầu file `graph.py` để ép console nhận mã hóa UTF-8: `sys.stdout.reconfigure(encoding='utf-8')`.

**Bằng chứng trước/sau:**
Dòng code khắc phục tại đầu file `graph.py`:
```python
if sys.stdout.encoding != 'utf-8':
    sys.stdout.reconfigure(encoding='utf-8')
```

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tôi làm tốt nhất ở khâu thiết kế kiến trúc hệ thống và xử lý lỗi môi trường. Việc chuyển đổi sang LangGraph giúp hệ thống có cấu trúc mạch lạc và chuyên nghiệp hơn rất nhiều so với bản Lab 08.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Tôi còn yếu ở việc tối ưu hóa hiệu năng (latency) của Supervisor. Hiện tại Supervisor phải gọi LLM hoàn toàn cho mỗi request, dẫn đến độ trễ trung bình khoảng 1-2 giây cho mỗi bước routing.

**Nhóm phụ thuộc vào tôi ở đâu?**
Nhóm phụ thuộc hoàn toàn vào khung sườn `graph.py` và `AgentState` mà tôi xây dựng. Nếu phần điều hướng này bị lỗi, các Worker con dù hoàn thiện đến đâu cũng không nhận được task để xử lý.

**Phần tôi phụ thuộc vào thành viên khác:**
Tôi phụ thuộc vào logic xử lý chi tiết bên trong các file `workers/`. Tôi cần các thành viên khác hoàn thiện hàm `run()` của từng Agent để pipeline có thể trả về câu trả lời thật thay vì các [PLACEHOLDER].

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ tập trung vào việc cải thiện **Supervisor Prompt** bằng kỹ thuật **Few-shot Prompting**. Qua kiểm tra các file trace (ví dụ `run_20260414_152633.json`), tôi nhận thấy LLM đôi khi vẫn mất thời gian phân vân giữa `retrieval_worker` và `policy_tool_worker` cho các ca "cấp quyền". Nếu có thêm thời gian, tôi sẽ đưa vào các ví dụ mẫu cụ thể trong prompt để Supervisor ra quyết định nhanh và chính xác hơn nữa.


---

*Lưu file này với tên: `reports/individual/[ten_ban].md`*  
*Ví dụ: `reports/individual/nguyen_van_a.md`*
