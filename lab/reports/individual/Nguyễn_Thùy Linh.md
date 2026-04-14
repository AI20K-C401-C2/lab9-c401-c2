# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Thùy Linh  
**Vai trò trong nhóm:** Docs Owner  
**Ngày nộp:** 14/04/2026

---

## 1. Tôi phụ trách phần nào?

Trong lab Day 09, tôi phụ trách chính mảng tài liệu kỹ thuật và tổng hợp báo cáo nhóm. Cụ thể, tôi trực tiếp hoàn thiện file `docs/system_architecture.md` và chịu trách nhiệm viết narrative cho `reports/group_report.md` dựa trên trace chạy thật. Tôi không trực tiếp viết hàm trong worker, nhưng tôi đóng vai trò kiểm chứng chéo giữa code và docs để đảm bảo mô tả kiến trúc không bị lệch với implementation. 

Song song đó, tôi hỗ trợ team về Git: thống nhất cách đặt nhánh, tách commit theo hạng mục (code/docs/report), nhắc mốc nộp trước 18:00 cho các file bị khóa theo `SCORING.md`, và xử lý các xung đột markdown khi nhiều người cùng sửa thư mục `docs/` và `reports/`. Việc này giúp các bạn code tập trung vào worker/eval, còn phần đầu ra tài liệu luôn bám sát trạng thái mới nhất của hệ thống.

Bằng chứng rõ nhất là phiên bản đã hoàn thiện của `docs/system_architecture.md`, trong đó tôi cập nhật đầy đủ pipeline, state schema, vai trò từng thành phần và các giới hạn thực tế dựa trên trace ở `artifacts/traces/`.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì?

Quyết định kỹ thuật quan trọng nhất của tôi là tài liệu hóa theo hướng “as-built”, tức mô tả đúng những gì hệ thống đang chạy, thay vì mô tả theo thiết kế lý tưởng của đề bài. Tôi cân nhắc hai phương án: 
1) điền docs theo form đẹp và đúng mục tiêu ban đầu; 
2) bám chặt code + trace thật để mô tả đúng hành vi hiện tại. 

Tôi chọn phương án 2 vì tiêu chí chấm nhấn mạnh bằng chứng thực tế từ trace, và `SCORING.md` cũng nêu rõ rủi ro mất điểm nếu report không khớp code. Khi rà trace, tôi thấy có nhiều điểm cần ghi trung thực: một số câu route vào `policy_tool_worker` nhưng `search_kb` trả rỗng; nhánh `human_review` đã trigger nhưng vẫn auto-approve; kết quả cuối cùng có abstain với confidence thấp. Nếu tôi bỏ qua các chi tiết này để viết theo “phiên bản lý tưởng”, nhóm sẽ khó debug và dễ mâu thuẫn khi bị hỏi trực tiếp.

Trade-off tôi chấp nhận là tài liệu trông “thật” hơn là “đẹp”: tôi phải ghi cả hạn chế của hệ thống, không chỉ ưu điểm. Đổi lại, nhóm có một bản kiến trúc dùng được cho review kỹ thuật. 

Ví dụ bằng chứng tôi đã dùng:

```json
{
	"workers_called": ["human_review", "retrieval_worker", "synthesis_worker"],
	"hitl_triggered": true,
	"final_answer": "Không đủ thông tin trong tài liệu nội bộ."
}
```

Trích từ trace `artifacts/traces/run_20260414_174254.json`.

---

## 3. Tôi đã sửa một lỗi gì?

Lỗi tôi xử lý là lỗi ở lớp tài liệu kiến trúc: `docs/system_architecture.md` ban đầu còn ở trạng thái template, rất nhiều phần để trống bằng placeholder. Symptom là nhóm không thể dùng tài liệu này để giải thích pipeline khi đối chiếu với trace, đặc biệt ở các mục “vai trò từng thành phần”, “shared state schema” và “giới hạn hệ thống”. Điều này không làm pipeline crash, nhưng tạo rủi ro mất điểm trực tiếp ở Group Documentation và làm cả nhóm khó thống nhất khi review.

Root cause là quá trình làm docs trước đó chỉ dừng ở mức khung, chưa có vòng đối chiếu bắt buộc với `graph.py`, `workers/*.py`, `mcp_server.py` và trace trong `artifacts/traces/`. Tôi đã sửa bằng cách đọc lại toàn bộ code cốt lõi, lấy dữ liệu từ các run thực tế (đặc biệt các run có route khác nhau), rồi viết lại file kiến trúc theo bản đầy đủ: overview, sơ đồ Mermaid, bảng thành phần, bảng state, so sánh single vs multi, và phần hạn chế cần cải tiến.

Trước khi sửa: file còn nhiều dòng “_________________”.  
Sau khi sửa: file đã có đầy đủ nội dung kiểm chứng được, bao gồm các field thực trong state như `risk_high`, `needs_tool`, `hitl_triggered`, `mcp_tools_used`, `latency_ms`, và `worker_io_logs`.

---

## 4. Tôi tự đánh giá đóng góp của mình

Điểm tôi làm tốt nhất là khả năng chuẩn hóa thông tin kỹ thuật thành tài liệu có thể kiểm chứng, giúp nhóm trình bày nhất quán giữa code, trace và report. Tôi cũng hỗ trợ Git tương đối hiệu quả trong giai đoạn nhiều file markdown được sửa song song, giảm xung đột khi merge.

Điểm tôi chưa tốt là vẫn làm kiểm tra consistency theo cách thủ công, nên tốn thời gian và dễ phụ thuộc vào việc các bạn gửi trace đúng lúc. Nếu được làm lại, tôi sẽ chuẩn bị checklist review docs từ sớm hơn thay vì dồn gần deadline.

Nhóm phụ thuộc vào tôi ở phần hoàn thiện các deliverable tài liệu (`docs/system_architecture.md`, `reports/group_report.md`) và kiểm soát tính khớp giữa báo cáo với implementation. Tôi phụ thuộc vào các bạn phụ trách worker/eval để có trace mới nhất, vì thiếu dữ liệu runtime thì tôi không thể viết phần phân tích trung thực.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì?

Nếu có thêm 2 giờ, tôi sẽ làm một script kiểm tra tự động độ khớp giữa trace và docs routing/comparison (ví dụ đối chiếu `supervisor_route`, `workers_called`, `mcp_tools_used`, confidence theo từng run). Tôi chọn cải tiến này vì các trace như `run_20260414_174052.json` cho thấy có run route đúng nhưng retrieval rỗng, dẫn đến abstain; nếu có dashboard cảnh báo sớm, nhóm sẽ sửa đúng chỗ nhanh hơn trước khi chốt báo cáo cuối.
