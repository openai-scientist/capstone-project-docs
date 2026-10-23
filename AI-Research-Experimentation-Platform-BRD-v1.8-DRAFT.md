# Business Requirements Document (BRD)
## AI Research Experimentation Platform

**Document Type:** Business Requirements Document\
**Project:** AI Research Experimentation Platform (codebase `logos` + `ai-research-platform-fe`)\
**Version:** 2.0 Draft (2026-10-23)\
**Status:** Viết lại theo hệ thống đã hiện thực. Thay thế toàn bộ BRD v1.x (dựa trên Popper target architecture).\
**Liên quan:** [PRD](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md) · [Architecture](architecture.md) · [changelog](changelog.md)

---

# 1. Executive Summary

AI Research Experimentation Platform giúp researcher đi từ **một chủ đề nghiên cứu hoặc một bảng dữ liệu** tới **một tập hypothesis có bằng chứng** và **một experiment plan đã được review độc lập**, với AI agent làm phần lớn công việc và con người kiểm soát ở các điểm quyết định.

Luồng nghiệp vụ:

```text
Topic (+ domains)            hoặc     Dataset version (+ objective tuỳ chọn)
        │                                        │
        │                                  [dataset] data card + research brief
        ▼                                        ▼
[hypothesis] literature → evidence cards → research gaps → hypotheses (falsifiable)
        ▼
[experiment_design] resources → plan → critique độc lập → final check → locked plan
        ▼
Researcher / AI PI: accept từng stage, send-back khi cần, đóng run
```

Ba giá trị cốt lõi:

1. **Traceability**: mỗi claim trỏ về record lưu trữ (paper, evidence card, cột dữ liệu); mọi con số do code tính.
2. **Review độc lập**: mỗi stage có reviewer riêng (model khác role `review`) và vòng revise có giới hạn; finding được công bố cùng artifact.
3. **Human control có thể chọn mức**: mode `light` để researcher trả lời gate; mode `auto` để AI Principal Investigator (PI) quyết thay trong giới hạn charter, vẫn escalate khi có vấn đề integrity.

Hệ thống **không** chạy experiment, không tính kết quả nghiên cứu để kết luận, không viết paper.

# 2. Business Context

Nghiên cứu dựa trên dữ liệu và literature cần nhiều bước thủ công: tìm và sàng lọc paper, trích bằng chứng, tìm gap, đặt hypothesis có thể bác bỏ, thiết kế experiment có baseline, control, power và kế hoạch phân tích. Các bước này tốn thời gian, đòi hỏi chuyên môn phương pháp luận và dễ sai mà không ai phát hiện.

LLM có thể làm nhanh các bước này nhưng có rủi ro: bịa trích dẫn, bịa số, trình bày giả định như kết luận, bỏ qua leakage hay baseline. Doanh nghiệp/nhóm nghiên cứu cần một nền tảng vừa tận dụng AI vừa giữ được provenance, review và quyền quyết định của con người.

# 3. Business Problems

| ID | Vấn đề |
|---|---|
| BP-01 | Tổng quan literature và tìm research gap tốn nhiều tuần và phụ thuộc người làm. |
| BP-02 | Hypothesis do người hay AI viết thường mơ hồ, không có tiêu chí bác bỏ. |
| BP-03 | Experiment plan thiếu baseline, control, phân tích thống kê, kiểm soát leakage; lỗi chỉ lộ ra sau khi đã chạy. |
| BP-04 | AI có thể bịa paper, số liệu hoặc nguồn; khó kiểm chứng nếu không có provenance. |
| BP-05 | Dataset thực tế khó hiểu nhanh (biến, kiểu, missing, mã hoá) và có rủi ro lộ dữ liệu khi đưa cho LLM. |
| BP-06 | Một AI tự chạy hết dễ tự xác nhận chính nó; cần reviewer độc lập và con người ở điểm rủi ro. |
| BP-07 | Chi phí LLM khó kiểm soát theo run và theo project. |
| BP-08 | Nhóm nghiên cứu cần cộng tác (phân quyền, comment, thông báo, audit) quanh cùng một run. |

# 4. Business Objectives

| ID | Mục tiêu | Liên kết |
|---|---|---|
| BO-01 | Rút ngắn đường từ topic/dataset tới hypothesis có bằng chứng và experiment plan có thể review. | BP-01, BP-02, BP-03 |
| BO-02 | Mọi claim, số liệu, nguồn đều truy được về record lưu trữ hoặc phép tính của code. | BP-04 |
| BO-03 | Hiểu dataset mà không để model thấy dòng dữ liệu. | BP-05 |
| BO-04 | Mỗi stage có review độc lập, finding công khai và revise có giới hạn. | BP-03, BP-06 |
| BO-05 | Con người giữ quyền ở gate và ở quyết định accept/send-back; mode auto có PI và escalation. | BP-06 |
| BO-06 | Kiểm soát chi phí bằng token budget theo run và budget tháng theo project. | BP-07 |
| BO-07 | Cộng tác an toàn: role cố định, lời mời, comment, thông báo, webhook, audit log. | BP-08 |
| BO-08 | Đo được chất lượng agent bằng bench có chấm điểm. | BP-06 |

# 5. Success Metrics

| Metric | Cách đo |
|---|---|
| Run hoàn tất | Tỉ lệ run tới `completed` so với `failed` (không tính `stopped` do người dùng). |
| Traceability | 0 artifact được publish với evidence ref không resolve (gate code chặn). |
| Chất lượng hypothesis | Điểm bench `logos_eval` trên hypothesis cases (ML/SE). |
| Chất lượng critic | Seeded-flaw recall của critic experiment_design (`python -m logos_eval.flaws`). |
| Tự phục hồi | Số stage được healing chạy lại thành công so với số cần người can thiệp. |
| Chi phí | Token mỗi run, tỉ lệ run `budget_exceeded`. |
| Thời gian | Thời gian từ tạo run tới stage cuối được publish. |

# 6. Stakeholders

| Stakeholder | Role hệ thống | Mối quan tâm |
|---|---|---|
| System Administrator | `platform_admin` | Cấu hình model/key, engine settings, budget mặc định, user, giám sát, audit |
| Project Manager | `project_manager` | Project, thành viên, dữ liệu, run, đóng/hoàn tất project, abandon run |
| Researcher / Data Analyst | `researcher` | Dữ liệu, bắt đầu run, trả lời gate, accept/send-back, charter, directive |
| Reviewer / Stakeholder | `reviewer` | Đọc kết quả, comment |
| AI Principal Investigator | Service `oversight` | Quyết định/khuyến nghị pipeline theo charter |

# 7. As-Is (không có platform)

Researcher tự tìm paper qua công cụ tìm kiếm, ghi chú thủ công, tự viết hypothesis và plan trong tài liệu rời; review dựa vào đồng nghiệp khi có thời gian; dùng chatbot thì không có provenance, không giới hạn chi phí, không lưu vết quyết định.

# 8. To-Be

- Một workspace theo project: dữ liệu, tài liệu, thành viên, run, kết quả, comment.
- Run được điều phối qua pipeline engine có contract rõ; tiến trình hiển thị trực tiếp (Run studio).
- Mỗi stage publish artifact bất biến kèm review finding, revision diff và gap.
- Researcher hoặc AI PI quyết định accept/send-back; mọi quyết định được ghi.
- Chi phí bị chặn bởi budget; lỗi hạ tầng được tự phục hồi.

# 9. Scope

## 9.1 Trong phạm vi (đã hiện thực)

| Nhóm | Nội dung |
|---|---|
| Tài khoản | Đăng nhập không mật khẩu (mã 6 số / link), Google sign-in tuỳ chọn, onboarding, avatar, admin tạo/khoá user |
| Project | Tạo, sửa, archive/restore, complete/reopen; trạng thái suy ra tự động; không hard delete |
| Thành viên | 3 project role + platform admin; mời qua email, hết hạn 24 giờ |
| Dữ liệu | Dataset CSV/XLSX có version; data connection PostgreSQL, MySQL, BigQuery, Google Sheets, Google Drive, Prometheus, InfluxDB; project files |
| Run | Pipeline `default` (topic) và `dataset_first` (dataset); mode `light`/`auto`; gate, accept, send-back, rerun, pause/resume/stop/abandon, sync |
| Engine | `dataset`, `hypothesis`, `experiment_design` |
| AI PI | Charter, stage outcome, failure, close; Critic (tắt mặc định); directive; obligation |
| Chi phí | Token budget theo run, tăng budget để resume; budget tháng project; cost ledger; usage admin |
| Cộng tác | Comment trên run, thông báo + SSE, preference, webhook Slack/Teams/Discord, audit log |
| Quản trị | Model theo role, integration (TypeSafe), admin/engine settings, overview, usage, log monitoring |
| Đánh giá | Bench `logos_eval` |

## 9.2 Ngoài phạm vi

- Chạy experiment, huấn luyện model, tính kết quả thống kê để kết luận finding.
- Viết paper/manuscript; Verify/confirmation trên dữ liệu reserved (kiến trúc Popper cũ).
- Organization/multi-tenant; project là scope cao nhất.
- Permission table hay custom role.
- Engine `objective` và pipeline nghiên cứu observational của Logos v1 (đã park ở tag `logos-v1-parked`).
- `explore` (Data Formulator) ở production cho tới khi có sandbox docker.

# 10. Business Requirements

## 10.1 Truy cập và cộng tác

| ID | Yêu cầu |
|---|---|
| BR-01 | Người dùng đăng nhập bằng email không mật khẩu; tài khoản tạo khi đăng nhập lần đầu; Google sign-in khi được cấu hình. |
| BR-02 | Quyền chỉ dựa vào role: `platform_admin` hoặc `user` ở cấp hệ thống; `project_manager`, `researcher`, `reviewer` ở cấp project. Không phải member thì không thấy project. |
| BR-03 | Project luôn có ít nhất một Project Manager active. Project archived chỉ đọc. |
| BR-04 | Thêm thành viên là lời mời; chỉ thành member sau khi chấp nhận. |
| BR-05 | Mọi thay đổi được ghi audit log; Project Manager xem log project, Admin xem log toàn hệ thống. |
| BR-06 | Thành viên comment trên run; nhận thông báo realtime; có thể gửi event run ra Slack/Teams/Discord. |

## 10.2 Dữ liệu

| ID | Yêu cầu |
|---|---|
| BR-07 | Upload CSV hoặc XLSX (sheet đầu); UTF-8, header unique, độ rộng dòng đều, tối đa 50 MiB mặc định. |
| BR-08 | File mới là version mới; không ghi đè dữ liệu cũ. |
| BR-09 | Import từ data connection (bảng hoặc `SELECT`) thành dataset version; credential mã hoá, không trả về; chặn host private. |
| BR-10 | Engine `dataset` tạo data card kiểu SPSS và research brief; mọi số do code tính; model không thấy dòng dữ liệu; không copy dòng dữ liệu vào artifact; nhóm nhỏ hơn `min_cell` bị gộp. |
| BR-11 | Researcher review data card và chọn research question ở gate (mode light). |

## 10.3 Hypothesis

| ID | Yêu cầu |
|---|---|
| BR-12 | Từ topic (12–2000 ký tự) và tối đa 6 domain, hệ thống tìm literature thật, sàng lọc, trích evidence card ở mức abstract, tổng hợp gap và sinh hypothesis. |
| BR-13 | Mỗi paper có provenance; mỗi card/gap/hypothesis tham chiếu record lưu trữ; paper không chấm điểm bị loại có lý do. |
| BR-14 | Mỗi hypothesis có falsification criterion nêu quan sát cụ thể làm nó sai. |
| BR-15 | Novelty report được gắn nhãn là đánh giá heuristic. |
| BR-16 | Researcher duyệt shortlist (bỏ paper / yêu cầu tìm lại) và duyệt tập hypothesis (giữ/bỏ / comment để revise). |
| BR-17 | Thiếu model, credential hay literature thì run báo lỗi rõ, không sinh nội dung giả. |

## 10.4 Experiment design

| ID | Yêu cầu |
|---|---|
| BR-18 | Từ hypothesis đã accept, hệ thống tìm resource có nguồn kiểm chứng (dataset, benchmark, baseline, code). |
| BR-19 | Hệ thống viết experiment plan phủ mọi hypothesis, mỗi hypothesis có claim và decision rule riêng; chỉ researcher được loại/hoãn hypothesis. |
| BR-20 | Plan được hai critic độc lập (Methodologist, Skeptic) review, revise có giới hạn, rồi final check mới. |
| BR-21 | Mọi tính toán thống kê (ví dụ power) do calculator có test tham chiếu; không bịa số. |
| BR-22 | Plan được đóng băng theo byte kèm hash; trạng thái review, gap, assumption nằm ở metadata. Plan chưa review xong vẫn publish với gap/finding mở; publish không đồng nghĩa với chấp thuận hay quyền chạy experiment. |
| BR-23 | Researcher review resource và plan ở gate; comment ở plan gửi plan qua thêm một vòng critique. |

## 10.5 Điều phối run

| ID | Yêu cầu |
|---|---|
| BR-24 | Mỗi project chỉ có một run active. |
| BR-25 | Run đi qua các stage của pipeline; stage hoàn tất phải được accept trước khi sang stage sau; có thể send-back về stage trước (tối đa `max_iterations`) hoặc rerun. |
| BR-26 | Mode `light`: researcher trả lời gate và quyết định accept/send-back. Mode `auto`: không mở gate tuỳ chọn, PI quyết định trong giới hạn. |
| BR-27 | Câu hỏi của agent không chặn run: hệ thống chọn default, ghi lại và đánh dấu không phải ý định researcher. |
| BR-28 | Giới hạn (vòng revise, token) kết thúc stage sớm với gap được ghi, không làm mất kết quả tốt nhất hiện có. |
| BR-29 | Lỗi hạ tầng tạm thời được tự chạy lại; khi cần người, hệ thống nói rõ cần làm gì. |
| BR-30 | Người có quyền pause, resume, stop run; Project Manager abandon run bị kẹt. |

## 10.6 AI Principal Investigator

| ID | Yêu cầu |
|---|---|
| BR-31 | PI soạn charter (research question, objective, success criterion); researcher có thể sửa và thêm directive. |
| BR-32 | PI quyết định (auto) hoặc khuyến nghị (light) ở stage outcome và failure; luôn viết close report. |
| BR-33 | PI không được kết thúc run, không trả lời gate engine, không accept item bị chặn trừ khi ghi rõ là known risk. |
| BR-34 | Rủi ro được chấp nhận trở thành obligation mà stage sau phải xử lý; close report nêu mọi obligation mở. |
| BR-35 | Escalation về integrity, xung đột charter hoặc không quyết được luôn chuyển cho researcher. |

## 10.7 Chi phí và quản trị

| ID | Yêu cầu |
|---|---|
| BR-36 | Run có token budget (mặc định 4M, tối đa 20M); hết budget thì pause và chờ tăng budget. |
| BR-37 | Project có budget tháng (soft cap) và cảnh báo; usage hiển thị theo project và cho admin. |
| BR-38 | Model key do admin nhập (BYOK), mã hoá, chỉ gửi kèm request; engine không lưu key. |
| BR-39 | Admin chỉnh behaviour settings và settings từng engine qua giao diện, không qua environment. |
| BR-40 | Admin có overview, usage, log monitoring realtime và audit log. |

# 11. Business Rules

| ID | Quy tắc |
|---|---|
| RULE-01 | Gate kiểm tra traceability, không kiểm tra đúng sai: chỉ provenance/contract hỏng mới bị từ chối; vấn đề nội dung thành finding hoặc known risk. |
| RULE-02 | Số liệu trong plan và kết quả đến từ code, không từ text của LLM. |
| RULE-03 | Mỗi hard rule trong prompt có validator trong code. |
| RULE-04 | Artifact bất biến; mọi thay đổi là version mới. |
| RULE-05 | Review pass khi không còn finding `blocking`. |
| RULE-06 | Approve gate và accept stage là hai hành động riêng. |
| RULE-07 | Trạng thái project do hệ thống suy ra (`draft`, `data_ready`, `researching`, `needs_review`), trừ `completed` do Project Manager đặt; `archived` thay thế các trạng thái khác. |
| RULE-08 | Platform lưu metadata và file, không đọc hay đánh giá nội dung khoa học. |

# 12. Assumptions & Constraints

- Có LLM gateway tương thích OpenAI và key hợp lệ; model role `review` nên khác model `stage` để review độc lập thật.
- Literature lấy từ provider công khai (OpenAlex…); card chỉ ở mức abstract.
- Một replica API hoặc volume dùng chung; PostgreSQL 16; storage local hoặc R2.
- Rate limit tính theo process; cần rate limit ở ingress khi scale.

# 13. Risks

| Risk | Giảm thiểu |
|---|---|
| LLM bịa nguồn/số | Evidence ref bắt buộc, gate traceability, calculator |
| Reviewer cùng model với tác giả | Role `review` cấu hình riêng; Critic PI riêng |
| Lộ dữ liệu cho model | Model chỉ thấy data card; `explore` tắt |
| Chi phí vượt | Token budget run + budget tháng project |
| Provider outage | Retry + pause `ProviderUnavailable`, self-healing |
| Auto mode quyết sai | Guard trong code, escalation bắt buộc, obligation, audit |
