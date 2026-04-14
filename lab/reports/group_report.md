# Báo Cáo Nhóm — Lab Day 09: Multi-Agent Orchestration

**Tên nhóm:** 
lab9-c401-c2 
**Thành viên:**
Nguyễn Thùy Linh ,
Bùi Minh Ngọc, 
Lê Đức Thanh,
Phạm Việt Anh,
Phạm Việt Hoàng ,
Phạm Đình Trường,
Phan Tuấn Minh 




| Tên | Vai trò | Email |
|-----|---------|-------|
| Việt Anh | Supervisor Owner | N/A |
| Minh ,Thanh| Worker Owner | N/A |
| Hoàng | MCP Owner | N/A |
| Linh, Ngọc, Trường | Trace & Docs Owner | N/A |

**Ngày nộp:** 14/04/2026  
**Repo:** https://github.com/AI20K-C401-C2/lab9-c401-c2
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Hướng dẫn nộp group report:**
> 
> - File này nộp tại: `reports/group_report.md`
> - Deadline: Được phép commit **sau 18:00** (xem SCORING.md)
> - Tập trung vào **quyết định kỹ thuật cấp nhóm** — không trùng lặp với individual reports
> - Phải có **bằng chứng từ code/trace** — không mô tả chung chung
> - Mỗi mục phải có ít nhất 1 ví dụ cụ thể từ code hoặc trace thực tế của nhóm

---

## 1. Kiến trúc nhóm đã xây dựng (150–200 từ)

> Mô tả ngắn gọn hệ thống nhóm: bao nhiêu workers, routing logic hoạt động thế nào,
> MCP tools nào được tích hợp. Dùng kết quả từ `docs/system_architecture.md`.

**Hệ thống tổng quan:**

Nhóm triển khai kiến trúc Supervisor-Worker với 4 node chính trong graph: supervisor, retrieval worker, policy tool worker, synthesis worker; ngoài ra có nhánh human review cho tình huống rủi ro cao. Luồng xử lý thực tế là input task đi vào supervisor để phân loại route, sau đó đi đến worker phù hợp, rồi hợp nhất tại synthesis để trả về final answer, confidence và trace metadata. Tất cả run đều được ghi ra artifacts/traces theo từng file json. 

Trong triển khai hiện tại, retrieval worker truy vấn ChromaDB collection day09_docs, policy worker chịu trách nhiệm kiểm tra exception và gọi tool qua MCP mock server, synthesis worker dùng grounded prompt để tạo câu trả lời hoặc abstain khi thiếu context. Hệ thống đã có đủ các thành phần theo yêu cầu sprint, nhưng chất lượng câu trả lời còn phụ thuộc mạnh vào trạng thái index retrieval.

**Routing logic cốt lõi:**
> Mô tả logic supervisor dùng để quyết định route (keyword matching, LLM classifier, rule-based, v.v.)

Supervisor sử dụng LLM classifier để trả về route, reason, risk_high, needs_tool dưới dạng JSON, thay vì keyword hardcode hoàn toàn. Khi route vào policy_tool_worker, hệ thống ghi rõ needs_tool và lý do route vào trace. Với case mơ hồ hoặc độ rủi ro cao, supervisor có thể đẩy sang human_review, sau đó tạm thời auto-approve để tiếp tục pipeline trong lab mode. Phần route_reason là tín hiệu chính để debug, vì cho thấy vì sao hệ thống chọn policy hay retrieval cho từng câu.

**MCP tools đã tích hợp:**
> Liệt kê tools đã implement và 1 ví dụ trace có gọi MCP tool.

- `search_kb`: ___________________
- `get_ticket_info`: ___________________
- `check_access_permission`: đã implement trong MCP server để xử lý cấp quyền Level 1/2/3 và emergency override.

- `search_kb`: tool truy vấn knowledge base, đầu vào query + top_k, đầu ra chunks/sources/total_found. Ví dụ trace gq02 và gq10 có gọi search_kb.
- `get_ticket_info`: tool tra ticket mock (ví dụ P1-LATEST), được gọi ở các câu có từ khóa ticket, p1, jira (ví dụ gq03, gq07).

---

## 2. Quyết định kỹ thuật quan trọng nhất (200–250 từ)

> Chọn **1 quyết định thiết kế** mà nhóm thảo luận và đánh đổi nhiều nhất.
> Phải có: (a) vấn đề gặp phải, (b) các phương án cân nhắc, (c) lý do chọn phương án đã chọn.

**Quyết định:** Nhóm chọn tách rõ policy logic và external capability vào policy_tool_worker + MCP server, thay vì để retrieval worker hoặc supervisor gọi trực tiếp dữ liệu/ticket API.

**Bối cảnh vấn đề:**

Trong sprint đầu, nhóm gặp vấn đề kiến trúc khi nhiều trách nhiệm bị dồn vào một điểm: route, retrieval, policy check, và tool call đan vào nhau khiến trace khó đọc. Khi câu trả lời sai, team không biết lỗi đến từ route sai, retrieval rỗng, hay policy áp dụng sai. Nếu tiếp tục để một worker xử lý hết, việc test độc lập theo contract sẽ không rõ ranh giới.

**Các phương án đã cân nhắc:**

| Phương án | Ưu điểm | Nhược điểm |
|-----------|---------|-----------|
| Gom policy + tool call vào retrieval worker | Ít file, triển khai nhanh ban đầu | Khó debug, contract retrieval bị phình, khó mở rộng tool |
| Tách policy_tool_worker và gọi qua MCP dispatch | Phân tầng rõ trách nhiệm, trace rõ mcp_tools_used | Tăng số node, tăng latency do thêm bước gọi tool |

**Phương án đã chọn và lý do:**

Nhóm chọn phương án tách policy_tool_worker và dùng MCP dispatch vì ưu tiên khả năng kiểm soát pipeline theo rubric Day 09 (route_reason, workers_called, mcp_tools_used). Nhờ tách lớp này, team có thể test retrieval độc lập với policy, và test policy độc lập với synthesis. Quyết định này cũng giúp mở rộng nhanh các tool mới như check_access_permission và create_ticket mà không phải sửa lõi orchestration trong graph.py.

**Bằng chứng từ trace/code:**
> Dẫn chứng cụ thể (VD: route_reason trong trace, đoạn code, v.v.)

```
# Trích grading_run.jsonl (gq03)
"supervisor_route": "policy_tool_worker",
"workers_called": ["policy_tool_worker", "synthesis_worker"],
"mcp_tools_used": ["search_kb", "get_ticket_info"]

# Trích grading_run.jsonl (gq09)
"workers_called": ["human_review", "retrieval_worker", "synthesis_worker"],
"hitl_triggered": true
```

---

## 3. Kết quả grading questions (150–200 từ)

> Sau khi chạy pipeline với grading_questions.json (public lúc 17:00):
> - Nhóm đạt bao nhiêu điểm raw?
> - Câu nào pipeline xử lý tốt nhất?
> - Câu nào pipeline fail hoặc gặp khó khăn?

**Tổng điểm raw ước tính:** 18 / 96 (ước tính nội bộ, chưa phải điểm chấm chính thức của giảng viên)

**Câu pipeline xử lý tốt nhất:**
- ID: gq10 — Lý do tốt: hệ thống trả lời đúng exception Flash Sale không hoàn tiền, confidence 0.95, route đúng vào policy_tool_worker và có gọi search_kb.

**Câu pipeline fail hoặc partial:**
- ID: gq01 (và nhiều câu retrieval khác) — Fail ở đâu: trả về abstain dù câu hỏi có đáp án trong bộ docs.  
  Root cause: retrieval không trả evidence (sources rỗng), kéo theo synthesis không có ngữ cảnh để trả lời.

**Câu gq07 (abstain):** Nhóm xử lý thế nào?

Nhóm xử lý gq07 theo hướng an toàn: abstain, không bịa mức phạt tài chính khi không có chứng cứ trong tài liệu. Trong grading_run, câu gq07 trả "Không đủ thông tin trong tài liệu nội bộ", đây là chiến lược giảm rủi ro hallucination và phù hợp tiêu chí anti-hallucination của rubric.

**Câu gq09 (multi-hop khó nhất):** Trace ghi được 2 workers không? Kết quả thế nào?

Với gq09, trace có ghi đủ 2+ worker (thực tế là 3 worker vì có human_review): human_review -> retrieval_worker -> synthesis_worker, đồng thời hitl_triggered=true. Tuy nhiên kết quả cuối vẫn abstain do không có chunk hỗ trợ nên không đạt mục tiêu multi-hop đầy đủ. Đây là bằng chứng rằng routing/hitl hoạt động, nhưng retrieval coverage chưa đạt.

---

## 4. So sánh Day 08 vs Day 09 — Điều nhóm quan sát được (150–200 từ)

> Dựa vào `docs/single_vs_multi_comparison.md` — trích kết quả thực tế.

**Metric thay đổi rõ nhất (có số liệu):**

Metric thay đổi rõ nhất là routing visibility: từ trạng thái không có route trace ở mô hình cũ sang có đầy đủ supervisor_route, route_reason, workers_called ở Day 09. Với 15 trace test hiện tại, phân bố route là 9 policy_tool_worker và 6 retrieval_worker; số run dùng MCP là 8/15; hitl_triggered là 2/15; avg latency khoảng 8.8 giây/run; avg confidence khoảng 0.347.

**Điều nhóm bất ngờ nhất khi chuyển từ single sang multi-agent:**

Điều nhóm bất ngờ nhất là kiến trúc đã tách đúng theo lý thuyết nhưng chất lượng câu trả lời vẫn thấp nếu retrieval không có dữ liệu. Nghĩa là orchestration đúng chưa đủ, chất lượng index và khả năng retrieve evidence mới là yếu tố chặn chất lượng end-to-end.

**Trường hợp multi-agent KHÔNG giúp ích hoặc làm chậm hệ thống:**

Trường hợp multi-agent chưa giúp ích là các câu chỉ cần fact retrieval đơn giản nhưng vẫn đi qua route và worker chain dài hơn, làm tăng độ trễ mà không cải thiện answer quality nếu chunk rỗng. Ví dụ các câu gq01, gq05, gq08 đều có route hợp lý nhưng output vẫn abstain.

---

## 5. Phân công và đánh giá nhóm (100–150 từ)

> Đánh giá trung thực về quá trình làm việc nhóm.

**Phân công thực tế:**

| Thành viên | Phần đã làm | Sprint |
|------------|-------------|--------|
| Việt Anh | Thiết kế graph orchestration, supervisor node, route decision, tích hợp human review branch | 1 |
| Minh, Thanh | Xây retrieval/policy/synthesis workers theo contract, test độc lập từng worker | 2 |
| Hoàng | Xây MCP mock server, dispatch tools, kết nối policy worker với MCP tools | 3 |
| Linh, Ngọc, Trường | Chạy eval/trace, hoàn thiện docs, tổng hợp group report và hỗ trợ Git workflow | 4 |

**Điều nhóm làm tốt:**

Điều nhóm làm tốt là phân vai rõ theo sprint và giữ được trace đầu ra tương đối đầy đủ để phân tích sau chạy. Team cũng phối hợp tốt ở phần chuẩn hóa cấu trúc tài liệu và commit theo mốc deadline.

**Điều nhóm làm chưa tốt hoặc gặp vấn đề về phối hợp:**

Điểm chưa tốt là đồng bộ giữa tiến độ code và chất lượng dữ liệu retrieval chưa theo kịp nhau. Nhóm hoàn thành khung đa agent đúng hạn nhưng thiếu bước xác nhận index/collection sớm, dẫn đến nhiều run abstain vào cuối buổi và ảnh hưởng điểm grading.

**Nếu làm lại, nhóm sẽ thay đổi gì trong cách tổ chức?**

Nếu làm lại, nhóm sẽ thêm một checkpoint kỹ thuật sau Sprint 2: bắt buộc xác minh retrieval trả chunk thật cho ít nhất 5 câu mẫu trước khi chuyển sang tối ưu routing/MCP, đồng thời thêm dashboard mini theo dõi tỉ lệ sources rỗng.

---

## 6. Nếu có thêm 1 ngày, nhóm sẽ làm gì? (50–100 từ)

> 1–2 cải tiến cụ thể với lý do có bằng chứng từ trace/scorecard.

Nếu có thêm 1 ngày, nhóm sẽ ưu tiên một cải tiến: tái xây index và thêm kiểm thử tự động cho retrieval coverage theo từng category câu hỏi (SLA, refund, access, HR). Lý do là trace hiện tại cho thấy nhiều câu route đúng nhưng vẫn không có evidence, khiến synthesis phải abstain. Sau khi retrieval ổn định, nhóm sẽ tune lại routing threshold để giảm over-route sang policy_tool_worker ở các câu truy vấn fact đơn giản.

---

*File này lưu tại: `reports/group_report.md`*  
*Commit sau 18:00 được phép theo SCORING.md*
