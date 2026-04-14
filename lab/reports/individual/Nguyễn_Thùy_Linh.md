# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Thùy Linh   
**Vai trò trong nhóm:** Docs Owner  
**Ngày nộp:** 14/04/2026 
**Độ dài yêu cầu:** 500–800 từ

---



---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- File chính: `docs/system_architecture.md`, `reports/group_report.md`, và phối hợp cập nhật `docs/routing_decisions.md`
- Functions tôi implement: không trực tiếp viết function Python trong worker, nhưng tôi chịu trách nhiệm chuẩn hóa schema tài liệu theo đúng implementation thật từ `graph.py`, `workers/*.py`, `mcp_server.py`, và trace trong `artifacts/traces/*.json`

**Cách công việc của tôi kết nối với phần của thành viên khác:**

Tôi đóng vai trò cầu nối giữa team code và team report. Khi Supervisor/Worker Owner thay đổi logic route, state field hoặc tool call, tôi đọc lại code rồi cập nhật tài liệu để không lệch giữa "thiết kế" và "thực thi". Song song, tôi hỗ trợ team dùng Git theo checklist chung: kéo nhánh mới trước khi sửa docs, tách commit theo sprint deliverable, và ưu tiên merge phần docs trước 18:00 theo đúng quy định trong `SCORING.md`. Việc này giúp nhóm tránh tình trạng xung đột khi nhiều bạn cùng sửa file markdown ở `docs/` và `reports/`.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**

Tôi có bằng chứng trực tiếp ở file `docs/system_architecture.md` (đã điền toàn bộ từ template trống thành bản v1.1 có sơ đồ pipeline, state schema và giới hạn hệ thống). Ngoài ra, phần mô tả kiến trúc của tôi bám sát trace thật, ví dụ các run `run_20260414_174052.json`, `run_20260414_174102.json`, `run_20260414_174254.json` trong `artifacts/traces/`, nơi thể hiện rõ route, workers_called, confidence và hitl_triggered.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi chọn tài liệu hóa kiến trúc theo "as-built architecture" (những gì hệ thống đang chạy thật) thay vì chỉ điền theo template/ý định ban đầu của lab.



**Lý do:**

Trong lúc hoàn thiện `docs/system_architecture.md`, tôi đứng trước hai lựa chọn. Lựa chọn 1 là điền theo thiết kế lý tưởng của đề bài: retrieval luôn trả evidence, policy luôn bám đúng domain, và HITL là quy trình dừng thật. Lựa chọn 2 là mô tả đúng hiện trạng chạy thực tế dựa trên code + trace. Tôi chọn lựa chọn 2 vì tiêu chí chấm của phần documentation nhấn mạnh tính kiểm chứng được bằng trace thực. Nếu chỉ viết "đẹp" theo mục tiêu ban đầu, report có thể mâu thuẫn với log thật và bị trừ điểm "report không khớp code/trace". Vì vậy tôi chủ động đưa vào tài liệu các điểm đang tồn tại thật như: `human_review` hiện là placeholder auto-approve, nhiều truy vấn rơi vào abstain khi `search_kb` trả chunks rỗng, và state thực tế có thêm `worker_io_logs` ngoài danh sách tối thiểu.

**Trade-off đã chấp nhận:**

Trade-off tôi chấp nhận là tài liệu sẽ bộc lộ các hạn chế của hệ thống hiện tại, nhìn "kém đẹp" hơn một bản mô tả lý tưởng. Tuy nhiên tôi ưu tiên tính trung thực kỹ thuật để team debug nhanh hơn và không bị lệch khi demo. Nhờ cách này, team có thể nhìn ngay rủi ro trọng yếu (index ChromaDB rỗng, route policy quá rộng, HITL chưa thực sự tương tác người dùng) thay vì tưởng mọi thứ đã hoàn chỉnh.

**Bằng chứng từ trace/code:**

```
# Trích trace run_20260414_174254.json
"workers_called": [
	"human_review",
	"retrieval_worker",
	"synthesis_worker"
],
"hitl_triggered": true,
"final_answer": "Không đủ thông tin trong tài liệu nội bộ.",
"latency_ms": 4130

# Trích trace run_20260414_174052.json
"supervisor_route": "policy_tool_worker",
"mcp_tools_used": [
	{
		"tool": "search_kb",
		"output": {
			"chunks": [],
			"sources": [],
			"total_found": 0
		}
	}
]
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)



**Lỗi:** Tài liệu kiến trúc ban đầu để nguyên dạng template, thiếu thông tin implementation thật nên không dùng được để debug hoặc đối chiếu rubric.

**Symptom (pipeline làm gì sai?):**

Symptom rõ nhất là khi team so khớp trace với docs, nhiều mục quan trọng không có câu trả lời: shared state chưa liệt kê đủ field, vai trò từng worker để trống, chưa có sơ đồ luồng thực tế. Nếu giữ nguyên như vậy, nhóm sẽ mất điểm trực tiếp ở mục Group Documentation (đặc biệt phần system architecture) và gây hiểu nhầm giữa các thành viên khi review pipeline.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**

Root cause nằm ở quá trình làm tài liệu trước đó chỉ bám template của lab mà chưa chạy vòng đối chiếu với code và trace sinh ra từ `graph.py`. Nói cách khác, lỗi không nằm ở worker logic mà nằm ở lớp "documentation contract" giữa implementation và báo cáo.

**Cách sửa:**

Cách sửa của tôi là đọc lại toàn bộ file kỹ thuật chính (`graph.py`, `workers/retrieval.py`, `workers/policy_tool.py`, `workers/synthesis.py`, `mcp_server.py`, `contracts/worker_contracts.yaml`, `eval_trace.py`) rồi trích bằng chứng từ trace thực. Sau đó tôi viết lại `docs/system_architecture.md` theo cấu trúc đầy đủ: overview, sơ đồ Mermaid, vai trò thành phần, shared state schema, so sánh single vs multi, và danh sách giới hạn cần cải tiến. Tôi cũng thống nhất cách ghi nhận thay đổi để team commit docs sạch, tránh sửa đè.

**Bằng chứng trước/sau:**
> Dán trace/log/output trước khi sửa và sau khi sửa.

Trước sửa: `docs/system_architecture.md` còn các placeholder như "_________________" ở hầu hết mục.

Sau sửa: file đã có nội dung triển khai thật, ví dụ liệt kê rõ các field state (`task`, `route_reason`, `risk_high`, `needs_tool`, `hitl_triggered`, `retrieved_chunks`, `policy_result`, `mcp_tools_used`, `final_answer`, `confidence`, `latency_ms`, `worker_io_logs`) và các giới hạn quan trọng như "human_review chưa là HITL thật".

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

> Trả lời trung thực — không phải để khen ngợi bản thân.

**Tôi làm tốt nhất ở điểm nào?**

Điểm tôi làm tốt nhất là khả năng "dịch" trạng thái kỹ thuật của hệ thống thành tài liệu có thể kiểm chứng. Tôi không chỉ mô tả lại đề bài mà bám vào artifact thực tế, nên các phần trong report nhóm nhất quán hơn khi thuyết trình. Ngoài ra, việc tôi hỗ trợ quy trình Git cho nhóm giúp tiến độ ổn định hơn ở giai đoạn cuối, nhất là lúc nhiều người cùng sửa markdown.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Điểm tôi làm chưa tốt là chưa tự động hóa được bước đối chiếu docs với trace. Tôi vẫn phải kiểm tra thủ công nên tốn thời gian, dễ bỏ sót nếu số lượng run tăng nhanh. Tôi cũng muốn chủ động hơn trong việc đề xuất checklist review chung sớm từ đầu buổi thay vì đợi gần deadline mới siết format.

**Nhóm phụ thuộc vào tôi ở đâu?** _(Phần nào của hệ thống bị block nếu tôi chưa xong?)_

Nhóm phụ thuộc vào tôi ở chỗ: nếu phần docs không hoàn chỉnh hoặc không khớp trace, nhóm có thể mất điểm ở cả phần Group Documentation và phần đánh giá mức đóng góp cá nhân. Ngoài ra, tôi giữ vai trò tổng hợp narrative cho `reports/group_report.md`, nên đây là đầu mối để nối code result với phần phân tích.

**Phần tôi phụ thuộc vào thành viên khác:** _(Tôi cần gì từ ai để tiếp tục được?)_

Tôi phụ thuộc vào thành viên phụ trách worker/eval để cung cấp trace chạy mới nhất và xác nhận những thay đổi logic quan trọng (route, policy exceptions, tool usage). Nếu thiếu dữ liệu này, tôi không thể viết báo cáo trung thực và đúng mốc thời gian của repo.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)


Nếu có thêm 2 giờ, tôi sẽ làm một script nhỏ để tự động kiểm tra độ nhất quán giữa trace và tài liệu routing/comparison, ưu tiên cho các câu khó kiểu multi-hop. Lý do là trace hiện tại cho thấy nhiều run có `search_kb` trả rỗng và answer abstain (ví dụ `run_20260414_174052.json`), nên nhóm cần dashboard nhỏ để thấy ngay tỉ lệ route đúng/sai và nguồn nào bị thiếu index trước khi nộp bản cuối.

---

*Lưu file này với tên: `reports/individual/[ten_ban].md`*  
*Ví dụ: `reports/individual/nguyen_van_a.md`*
