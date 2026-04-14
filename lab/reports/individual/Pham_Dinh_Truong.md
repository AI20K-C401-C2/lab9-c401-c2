# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Phạm Đinh Trường  
**Vai trò trong nhóm:** Trace Owner   
**MSV**: 23010136 

**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- File chính: Gỡ lỗi khởi chạy phần evaluation qua `eval_trace.py` và theo dõi toàn bộ các trace data chạy được xuất ra thư mục `artifacts/traces/`.
- Functions tôi implement/quản lý: Vận hành script `run_grading_questions()` và `analyze_traces()` để đánh giá toàn bộ multi-agent pipeline trong Lab. Tôi chịu trách nhiệm khắc phục lỗi môi trường local và đảm bảo pipeline output ra json log chính xác.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Tôi đánh giá kết quả tổng thể và sinh ra các file trace. Các thành viên đảm nhận phần tính năng như Worker, MCP (cụ thể là `search_kb`) sẽ phải dựa vào log đánh giá của tôi (đọc các metadata như `latency_ms`, `route_reason`) để biết module của mình chạy ra sao. Tôi có thể biết liệu supervisor có bắt đúng keyword để map tới công cụ MCP một cách chuẩn nhất hay không. Công việc của tôi đóng vai trò như QA và system analysis.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**
- Đã chạy thành công trace thực tế thu thập hôm nay, có output: `artifacts/traces/run_20260414_175412.json`
- Thực thi CLI đánh giá: `python eval_trace.py --grading` để sinh file lưu log JSONL.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** 
Trong luồng logging multi-agent pipeline, tôi quyết định cấu hình giữ nguyên các biến `route_reason` và `latency_ms` chi tiết trong JSON trace xuất ra (`grading_run.jsonl`) thay vì chỉ in ra console (stdout), đồng thời serialize output cho từng câu truy vấn dạng file JSON độc lập để làm log phân tích chuẩn.

**Lý do:**
Với hệ thống single-agent cũ (Day 08) thì luồng gọi khá tường minh. Nhưng sang Day 09, mô hình RAG có nhiều Worker (ví dụ: `policy_tool_worker`, `synthesis_worker`). Do Supervisor đứng ở giữa, việc lưu lại đầy đủ node hành trình + `route_reason` vào trace giúp giải thích minh bạch TẠI SAO Supervisor lại phân luồng đến Worker đó. Nếu chỉ ghi log lên terminal, chúng ta không thể nào xử lý phân tích số liệu tự động (batch analysis) được.

**Trade-off đã chấp nhận:**
Dữ liệu log phình ra thành rất nhiều file nhỏ trong `artifacts/traces/` kéo theo IO Overhead. Tuy nhiên I/O trên đĩa cứng chỉ chậm cỡ (milliseconds), gần như không đáng kể khi so sánh với hàng trăm phần nghìn giây API latency gọi tới model. Bù lại hệ thống debug cực kì rõ nét. 

**Bằng chứng từ trace/code:**

```json
// Trích xuất trong artifacts/traces/run_20260414_175412.json
{
  "task": "Khách hàng có thể yêu cầu hoàn tiền trong bao nhiêu ngày?",
  "supervisor_route": "policy_tool_worker",
  "route_reason": "Câu hỏi liên quan đến chính sách hoàn tiền cụ thể. | needs_tool=True, will use MCP",
  "latency_ms": 12151,
  "workers_called": [
    "policy_tool_worker",
    "synthesis_worker"
  ]
}
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Sự cố xung đột DLL (loading issue WinError 1114) và set up environment biến `.env` gây crash ngầm trên Windows.

**Symptom (pipeline làm gì sai?):**
Trong quá trình triển khai cấu hình ban đầu chạy scripts của hệ thống, pipeline bị sập đứng im ngay (silent crash) khi backend ChromaDB của indexer và evaluation script tìm cách nạp dll qua HNSWLib trên Windows Platform. Báo lỗi không catch được bằng Block try/except bình thường. Cộng thêm việc các Token gọi lên Gemini/OpenAI trượt vì tham chiếu `.env` nhiều chỗ sai kịch bản.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**
Lỗi nằm ở tầng System Dependency và Environment Configuration . Việc import library ChromaDB/PyTorch ở trên Windows bị kẹt ở bộ nạp dll WinError và việc định nghĩa môi trường ảo không rõ ràng từ đầu. API Key thiếu nhất quán khiến MCP tool không auth được để search KB.

**Cách sửa:**
- Xây dựng lại môi trường Python Environment sạch, cài dependency cẩn thận.
- Gom toàn bộ cấu hình Secret vào chung 1 file `.env` trung tâm.

**Bằng chứng trước/sau:**
> Trước khi cố định: Chạy script RAG lỗi màn hình sập và crash log Windows.
> Sau khi cấu hình môi trường chuẩn: Export được logs đúng định dạng:
```text
[supervisor] received task: Khách hàng có thể yêu cầu hoàn tiền trong bao nhiêu ngày?
[supervisor] route=policy_tool_worker reason=Câu hỏi liên quan đến chính sách hoàn tiền cụ thể. | needs_tool=True, will use MCP
[policy_tool_worker] called MCP search_kb
[graph] completed in 12151ms
```

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tôi kiên nhẫn bám trụ gỡ lỗi hệ sinh thái, khắc phục được các vấn đề crash trầm trọng trên nền Windows khi gọi thư viện. Qua đó, tôi khởi chạy được hoàn thiện script evaluation. File tổng kết này chứng minh cho việc system pipeline hôm nay của chúng ta đã work end-to-end.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Hệ thống xử lý còn chậm, hiện xử lý 1 trace truy vấn qua các tác vụ model ngốn ~12s (như trace 12151ms), việc này chưa tương xứng với kỳ vọng giảm độ trễ thực tế.

**Nhóm phụ thuộc vào tôi ở đâu?** _(Phần nào của hệ thống bị block nếu tôi chưa xong?)_
Nếu tôi không thể hoàn tất giải quyết môi trường và fix các script eval, team sẽ không thu lượm được `grading_run.jsonl`, tức là không có trace output gì cả. Từ đó 100% fail điểm Assignment đánh giá hệ thống.

**Phần tôi phụ thuộc vào thành viên khác:** _(Tôi cần gì từ ai để tiếp tục được?)_
Tôi cần các bạn phụ trách Worker và công cụ MCP phải tinh chỉnh System Prompt tối ưu để khi eval chạy hàng loạt thì nó ra response ổn định, tránh API lỗi hay Format Output sai khiến Script parsing JSON của tôi báo exception.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ tối ưu code script chạy test `eval_trace.py` bằng lập trình bất đồng bộ (`asyncio`). Từ report của `run_20260414_175412.json` ở Trace q02, tổng số Latency tốn tới hơn 12.1 giây. Nếu có hàng trăm câu truy vấn mà làm Process chạy Sequential sẽ cực kì lãng phí. Triển khai Asyncio giúp tôi bắn Request cho LLM/Worker song song, giảm được đáng kể thời gian test batch và lấy kết quả log nhanh hơn.
