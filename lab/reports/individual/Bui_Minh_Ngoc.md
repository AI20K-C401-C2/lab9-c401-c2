# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Bùi Minh Ngọc  
**Vai trò trong nhóm:** Docs Owner  
**Ngày nộp:** 2026-04-14  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- `docs/routing_decisions.md` — Tổng hợp và phân tích các quyết định routing từ trace thực tế
- `docs/single_vs_multi_comparison.md` — So sánh định lượng giữa kiến trúc Day 08 (single-agent) và Day 09 (multi-agent)

**Cách công việc của tôi kết nối với phần của thành viên khác:**

Tôi đọc toàn bộ trace files trong `artifacts/traces/` (60 files) và `artifacts/grading_run.jsonl` (10 câu grading) để phân tích routing distribution, accuracy, latency. Kết quả từ báo cáo routing giúp nhóm biết Supervisor đang route đúng hay sai, từ đó các thành viên phụ trách `graph.py` và workers có dữ liệu để tinh chỉnh prompt. Phần `single_vs_multi_comparison.md` tôi lấy số liệu Day 08 từ `lab8-C401-c2/results/scorecard_baseline.md` (faithfulness=3.80, completeness=3.50) để so sánh trực tiếp với kết quả Day 09 từ trace.

**Bằng chứng:**

`docs/routing_decisions.md` (3 routing decisions từ grading trace); `docs/single_vs_multi_comparison.md` (6 sections so sánh)

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Phân tích routing decisions theo **từng loại câu hỏi** (đơn giản, multi-hop, abstain) thay vì chỉ so sánh metrics trung bình.

**Lý do:**

Ban đầu tôi định chỉ tính trung bình chung (avg faithfulness, avg latency) rồi kết luận multi-agent tốt hơn hay kém hơn. Nhưng khi đọc trace, tôi nhận ra kết quả rất khác biệt theo loại câu: câu đơn giản (q01: SLA P1) Day 08 đã trả lời tốt (faithfulness=5/5), multi-agent không cải thiện thêm mà còn tăng latency. Ngược lại, câu abstain (q09: ERR-403-AUTH) Day 08 trả lời "Không đủ dữ liệu" với faithfulness=1/5, trong khi Day 09 Supervisor nhận diện được rủi ro và trigger HITL (`hitl_triggered: true`, `route_reason: "unknown error code + risk_high → human review"` — trace file `run_20260414_165327.json`).

**Trade-off đã chấp nhận:**

Phân tích theo loại câu tốn thời gian hơn (phải cross-reference từng trace với `test_questions.json`), nhưng cho kết luận chính xác hơn: multi-agent chỉ thực sự vượt trội ở câu multi-hop và abstain, không phải mọi loại câu.

**Bằng chứng từ trace/code:**

```
# single_vs_multi_comparison.md — Section 2:
# Câu đơn giản: Day 08 accuracy 90%+ vs Day 09 accuracy 95%+ (cải thiện nhỏ)
# Câu multi-hop: Day 08 accuracy 50% vs Day 09 accuracy 85%+ (cải thiện lớn)
# Câu abstain: Day 08 abstain_rate 30% vs Day 09 abstain_rate 10% (giảm nhờ HITL)
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Routing Distribution trong `routing_decisions.md` ban đầu ghi sai số liệu — tổng câu route không khớp với số trace thực tế.

**Symptom (tài liệu sai gì?):**

Khi tổng hợp routing distribution lần đầu, tôi đếm retrieval_worker=25, policy_tool_worker=31, human_review=6 (tổng=62) nhưng tổng số câu chỉ có 25 (15 test + 10 grading). Phần trăm cũng cộng vượt 100% (44%+55%+10%=109%). Nguyên nhân: tôi đếm nhầm vì mỗi câu qua HITL (human_review) sau đó được route tiếp sang retrieval_worker, nên bị đếm đôi ở cả hai worker.

**Root cause:**

Trace của câu HITL (VD: `run_20260414_165327.json`, câu q09) ghi `workers_called: ["human_review", "retrieval_worker", "synthesis_worker"]` — tức human_review rồi tiếp sang retrieval. Tôi đếm cả hai worker thay vì chỉ đếm worker đầu tiên do Supervisor chọn.

**Cách sửa:**

Đếm routing distribution theo `supervisor_route` (worker đầu tiên Supervisor chọn), không đếm theo `workers_called` (tất cả workers đã đi qua).

```
# TRƯỚC khi sửa:
retrieval_worker: 25 | policy_tool_worker: 31 | human_review: 6 → tổng 62 (sai)

# SAU khi sửa — đếm theo supervisor_route thực tế từ 25 traces:
retrieval_worker: 10/25 (40%) | policy_tool_worker: 13/25 (52%) | human_review: 2/25 (8%)
→ tổng = 25 ✅
```

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**

Phần phân tích so sánh trong `single_vs_multi_comparison.md` — có số liệu cụ thể từ cả hai lab: Day 08 baseline (faithfulness=3.80, completeness=3.50 theo `scorecard_baseline.md`) và Day 09 trace thực tế (avg confidence grading = 0.39, avg latency grading ≈ 13,146ms). Không ước đoán.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Phần Routing Decision #4 (bonus) trong `routing_decisions.md` chưa điền — tôi hết thời gian trước deadline 18:00 nên chưa bổ sung thêm case phân tích.

**Nhóm phụ thuộc vào tôi ở đâu?**

Nhóm dựa vào `routing_decisions.md` để biết Supervisor routing có chính xác không, và `single_vs_multi_comparison.md` để viết phần kết luận trong group report.

**Phần tôi phụ thuộc vào thành viên khác:**

Cần `graph.py` (Supervisor logic) và workers chạy được để có trace data. Cần `scorecard_baseline.md` từ Day 08 để so sánh.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ bổ sung **Routing Decision #4** trong `routing_decisions.md` bằng câu gq09 (P1 lúc 2am + cấp Level 2 access) — đây là câu multi-hop 16 điểm, trace cho thấy Supervisor trigger HITL (`hitl_triggered: true`) rồi route sang retrieval nhưng confidence chỉ 0.2 và answer = "Không đủ thông tin". Phân tích này cho thấy hạn chế: HITL fallback an toàn nhưng không giúp pipeline trả lời tốt hơn cho câu cross-document phức tạp. Đồng thời sẽ dọn sạch các placeholder `_________________` còn sót trong cả hai file docs.

---

*File: `reports/individual/Bui_Minh_Ngoc.md`*
