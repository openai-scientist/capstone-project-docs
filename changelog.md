# Changelog

Lịch sử thay đổi và registry của toàn bộ tài liệu canonical trong
`capstone-project-docs`. Các đường dẫn bên dưới là đường dẫn tương đối từ
repository root.

## 2026-10-23 — Viết lại BRD, PRD và architecture theo hệ thống đã code (v2.0 DRAFT)

### Changed

- [BRD](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md), [PRD](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md) và [architecture](architecture.md): viết lại toàn bộ thành v2.0. Bản cũ mô tả "Popper target architecture" (Understand ⇄ Ground ⇄ Discover ⇄ Verify ⇄ Communicate, Analysis Ledger, Verify trên dữ liệu reserved, manuscript), một kiến trúc không được hiện thực. Bản mới bám theo code đang chạy:
  - Platform `logos/services/management`: auth không mật khẩu, project và 3 project role, dataset, data connection, run, stage, gate, accept/send-back, self-healing, token budget, BYOK, admin, webhook, audit.
  - Engine `dataset`, `hypothesis`, `experiment_design` trên skeleton `logos_engine`; pipeline `default` và `dataset_first`; engine contract 3.3.0.
  - AI PI `logos/services/oversight` (charter, stage outcome, failure, close; Critic tắt mặc định; obligation).
  - Frontend `ai-research-platform-fe` (Next.js 16): route, section của workspace, Run studio.
- Giữ nguyên tên file để không làm hỏng liên kết; version mới ghi trong header của từng file.

### Validation

- Đối chiếu với code: README/AGENTS của từng project trong `logos`, `logos_contract.base.CONTRACT_VERSION`, `management/research/schemas/runs.py`, các router trong `management`, `research/pi/authority.py`, `research/platform_settings.py`, `integrations/catalog.py`, `core/roles.py`, `deploy/.env.example` (`ENGINES`, `PIPELINES`), `stages/*/engine.py`, các route `page.tsx` và `workspace-sections.tsx` của FE.
- `git diff --check` sạch. Liên kết giữa BRD, PRD, architecture và changelog đều trỏ tới file có thật.
- Không có diagram nào phải regenerate.

## 2026-10-06 — FE nối sẵn API cho luồng topic → hypothesis, chờ Platform BE (DRAFT)

### Changed

- [Platform BE handoff](be-handoff-topic-to-hypothesis.md), mục 13: đổi từ "việc FE sẽ làm" thành những gì FE đã làm. Gồm client cho mục 3.1–3.6, cách đọc lại event rồi nối SSE, cách mở lại run sau reload (`GET /runs?limit=1`, nên `GET /runs` phải sắp mới nhất trước và trả `topic`, `label`, `status`), và các việc còn lại khi BE xong.
- Repo FE: chọn nguồn run bằng `NEXT_PUBLIC_RUN_STREAM_SOURCE` (`mock` mặc định, hoặc `api`). Mock của topic → hypothesis giữ lại cho tới khi BE chạy qua checklist. Experiment, Interpret, Report và Write hiện trên UI ở dạng khoá.
- Không cần sửa BRD/PRD/ERD: chưa đổi requirement hay contract, chỉ nối FE theo tài liệu đã có.

### Validation

- Repo FE: `npx tsc --noEmit`; `npx biome check` không có lỗi trên các file đã sửa; `npx vitest run` 184 test pass, gồm 5 test mới cho SSE (`run-events.test.ts`).
- Playwright trên FE thật, chế độ `mock`: trọn luồng trên desktop và điện thoại, không lỗi console, so khớp với ảnh chụp trước khi sửa.
- Playwright trên FE thật, chế độ `api` với một BE giả theo mục 3.1–3.6: happy case, pause và resume, reload giữa chừng, gate sau reload, run fail giữa chừng, mở lại run đã xong. Không lỗi console, không request lỗi.

## 2026-10-06 — Handoff cho luồng topic → hypothesis: engine và Platform BE (DRAFT)

### Added

- [Engine handoff](engine-handoff-topic-to-hypothesis.md): cho team repo lõi chạy pipeline stage 1–8. Gồm ranh giới trách nhiệm với Platform BE; API engine cung cấp (`POST /runs` idempotent theo `platform_run_id`, đọc trạng thái, đọc lại event, trả lời gate, pause, resume, cancel); cách đẩy event về Platform BE với `source_seq` và chính sách retry; danh mục event chi tiết cho từng stage kèm validate; gate; quy tắc chia nhỏ event; 10 tình huống lỗi; cách test.
- [Platform BE handoff](be-handoff-topic-to-hypothesis.md): viết lại cho `ai-research-platform-be`. Platform BE vẫn là API duy nhất của FE: tạo run bằng `topic`, `domains`; đọc lại event; SSE theo mẫu notification stream; trả lời gate; pause, resume; sửa `sync` và `abandon`. Thêm endpoint nhận event từ engine, sửa `PopperClient`, pacer, happy case, 15 unhappy case, migration (`run_events`, `run_gates`, cột mới của `research_runs`), cách test và checklist.

### Changed

- [Research run event stream](research-run-event-stream.md): tách SSE sang `GET …/events/stream`; `GET …/events` trả JSON phân trang. `run.started.dataset` thành tuỳ chọn, vì run đi từ topic không có dataset. FE đã sửa `contract.ts` và mock tương ứng.
- Không cần sửa BRD/PRD/ERD trong đợt này: đây là tài liệu triển khai cho contract nháp, chưa đổi requirement.

### Validation

- `npx tsc --noEmit` và `npx vitest run src/lib/run-stream` trong repo FE: 9 test pass.
- Đối chiếu với `contract.ts`, `reducer.ts`, `mock-run.ts` (thứ tự event, payload, nhịp gửi), và với code Platform BE hiện tại: `api/v1/runs.py`, `api/v1/internal_popper.py`, `api/v1/notifications.py`, `services/popper_client.py`, `core/responses.py`, `core/config.py`.

## 2026-10-06 — Event stream: tên artifact cho người dùng, `StagePlan.reads` và phần còn thiếu của stage 1, 4, 7 (DRAFT)

### Changed

- [Research run event stream](research-run-event-stream.md): `StagePlanStep.artifact` nay là tên cho người dùng đọc (ví dụ "Research goal", "Gap map") thay vì đường dẫn như `stage-01/goal.md`; thêm `StagePlan.reads` tuỳ chọn để FE hiện stage dựa vào gì và viết ra gì.
- Bổ sung phần còn thiếu so với [hướng dẫn pipeline](../ai-research-platform-fe/docs/pipeline_stage_1_to_8_guide.md):
  - Stage 1: event `scope.approved` (`at`, `note`) cho bước PI duyệt goal, kèm mốc giờ lưu.
  - Stage 4: `literature.merged` đổi `sources[]` thành `records[]` (nguồn, ID, số trích dẫn, có DOI) cộng với `kept`. `literature.collected` thêm `collected_at`. `ShortlistPaper` thêm `source` và `citations`.
  - Stage 7: thêm event `synthesis.overview` và `synthesis.ranked` (cơ hội nghiên cứu xếp ưu tiên).
- Bỏ hiển thị skill trên UI. Event `skills.loaded` vẫn nằm trong contract.
- Lý do: màn hình run từ topic tới hypothesis không còn lộ tên file (`.md`, `.json`, `.jsonl`, `.yaml`) hay tên cột dữ liệu cho người dùng.
- Không cần sửa BRD/PRD/ERD: chỉ đổi cách hiển thị và một trường tuỳ chọn của contract nháp.

### Validation

- `npx tsc --noEmit`, `npx vitest run src/lib/run-stream` (9 test pass) và `npx biome check` (không có lỗi) trong repo FE.
- Chạy trọn luồng topic → hypothesis trên FE thật bằng Playwright, cả desktop và màn hình điện thoại: không có lỗi console.

## 2026-10-06 — Event stream: pipeline stage 1–8 thay cho Understand và Hypothesize (DRAFT)

### Changed

- [Research run event stream](research-run-event-stream.md): thay stage `understand` bằng 5 stage theo pipeline stage 1–7 ([hướng dẫn pipeline](../ai-research-platform-fe/docs/pipeline_stage_1_to_8_guide.md)): `scope` (1–2), `search` (3–4), `screen` (5), `read` (6), `synthesize` (7). `hypothesize` (stage 8) nay sinh hypothesis qua tranh luận 3 góc nhìn, mỗi hypothesis có 4 phần gồm tiêu chí bác bỏ `falsify`, kèm kiểm tra tính mới và khả thi.
- Khoá test plan (`plan.locked`, thêm `design`) chuyển sang step đầu của `experiment` (stage 9).
- Thêm event `skills.loaded`; thêm agent `strategist`, đổi `knowledge` thành `librarian`; thêm luật 5 (hypothesis phải bác bỏ được) và luật 6 (con số nhớ ra chưa kiểm chứng cho tới khi có bài báo xác nhận).
- Gate: `framing` đổi thành `scope`; thêm gate `screen` (HITL stage 5) cho Copilot và Full, cho phép bỏ bài khỏi shortlist.
- Lý do: luồng từ đầu tới sinh hypothesis của Popper đổi theo AutoResearchClaw stage 1–8; các stage từ Experiment về sau giữ nguyên.
- Run bắt đầu từ **topic** người dùng nhập, không còn dùng Research context: `run.started` đổi `question`, `context_version` thành `topic`, `domains[]`; body `POST …/runs` đề xuất nhận `topic`, `domains[]`. Sub-question do Popper tự sinh ở stage 2.
- Tạm thời run chỉ đi từ topic tới hypothesis (stage 1–8): ẩn bước chọn dataset và stage Check the data, ẩn bước chọn review mode (mặc định Copilot, chỉ dừng ở gate `screen`), và kết thúc ngay sau khi PI chọn bộ hypothesis. `run.completed` khi đó không có `paper_title`. Phần Experiment, Report, Paper vẫn giữ trong contract và code, bật lại bằng cờ trong `src/lib/run-stream/features.ts` của repo FE.
- Chưa sửa BRD/PRD/ERD, cùng lý do với entry bên dưới.

### Validation

- `npx tsc --noEmit` và `npx vitest run src/lib/run-stream` trong repo FE: 8 test pass, gồm run Copilot bỏ một bài ở shortlist và H4, và run Full kiểm tra `plan.locked` nằm trong `r1-experiment`, trước `node.created` đầu tiên.
- Chạy trọn luồng trên FE thật (Playwright, desktop và màn hình điện thoại, Copilot): không có lỗi console.

## 2026-10-06 — Contract event stream cho research run (DRAFT)

### Added

- [Research run event stream](research-run-event-stream.md): contract giữa Platform BE và FE cho workspace của research run. Nội dung gồm endpoint SSE `GET …/runs/{runId}/events` (resume bằng `Last-Event-ID`), lệnh trả lời gate, pause/resume, envelope event, toàn bộ danh mục event theo 7 stage (Data, Understand, Hypothesize, Experiment, Interpret, Report, Write), điểm dừng theo 4 review mode và cách ánh xạ với `research_runs.status`, `frame_reviews`, `run_artifacts` hiện có.
- Lý do: FE đã dựng workspace từ event (đang dùng mock phát đúng contract này), nên BE và Popper cần một spec chung để phát event thật.
- Chưa sửa BRD/PRD/ERD: contract đề xuất thêm `review_mode`, bảng event và gate tổng quát; các thay đổi này cần team BE xác nhận trước khi đưa vào tài liệu canonical.

### Validation

- Đối chiếu từng event với `ai-research-platform-fe/src/lib/run-stream/contract.ts`.
- `npx vitest run src/lib/run-stream` trong repo FE: 7 test pass, gồm phát trọn một run Auto và một run Copilot có bỏ hypothesis theo đúng thứ tự gate trong tài liệu.

## 2026-10-02 — Conceptual ERD: chỉ vẽ cho Platform BE

### Changed

- Thay Conceptual ERD chung của cả hệ thống bằng [Platform BE Conceptual ERD](diagrams/erd/conceptual-erd/Platform-BE-Conceptual-ERD.drawio), sinh từ [ERD model](diagrams/erd/conceptual-erd/platform_be_conceptual_erd.json): 13 entity, 19 relationship.
- Lý do: Popper v2 thay kiến trúc cũ nên nhóm entity core của Popper v1 không còn khớp, và phần Popper hiện chưa đủ rõ để vẽ. Sơ đồ chỉ gồm những gì Platform BE quản lý.
- Entity: `User`, `Role`, `Project`, `ProjectMember`, `Dataset`, `DatasetVersion`, `ResearchContext`, `Run`, `FrameReview`, `RunArtifact`, `Comment`, `Notification`, `AuditLog`.
- Sơ đồ theo chuẩn conceptual ERD: mỗi khái niệm nghiệp vụ xuất hiện đúng một lần, mọi relationship có tên và cardinality crow's-foot, không có thuộc tính, khoá hay chi tiết lưu trữ.
- Entity mới so với bản trước: `ResearchContext`, `FrameReview`, `RunArtifact`, `AuditLog`. `DatasetSnapshot` đổi tên thành `DatasetVersion` theo PRD §44.
- Bỏ `BillingAccount`, `PublicationRelease`, `PolicyVersion`, `DecisionRequest`, `EvaluationDecision`, `DatasetAssessment`, `UsageRecord` và toàn bộ entity core của Popper v1.
- Xoá file `.drawio` và model của ERD chung cũ; [builder](diagrams/erd/conceptual-erd/build_conceptual_erd.py) nay mặc định sinh ERD của Platform BE và in tên relationship lên đường nối (trước đây để trống).
- Quyết định nghiệp vụ đi kèm: Reviewer chỉ đọc và bình luận, không có bước ký duyệt; dataset đợt này chỉ nhận CSV.
- Chưa sửa BRD/PRD: hai tài liệu này vẫn tham chiếu kiến trúc Popper cũ (các điểm quyết định, Verify, publication view) và cần một đợt đồng bộ riêng với Popper v2.

### Validation

- `python3 diagrams/erd/conceptual-erd/build_conceptual_erd.py`: `entities=13 relationships=19 crossings=0`, không có lỗi routing.
- Export PNG bằng draw.io desktop và kiểm tra bằng mắt: không có đường đè lên entity, marker chồng nhau hay nhãn che nhau.

## 2026-10-02 — Conceptual ERD: bỏ Organization, Project là scope cao nhất

### Changed

- Cập nhật [Conceptual ERD](diagrams/erd/conceptual-erd/AI-Research-Experimentation-Platform-Conceptual-ERD.drawio) và [ERD model](diagrams/erd/conceptual-erd/conceptual_erd.json): còn 42 entity, 69 relationship.
- Bỏ entity `Organization` và ba relationship của nó (`owns` Project, `has member` User, `billed through` BillingAccount). BRD (BR-03, BRule-01) và PRD (§9, §10 FR-AUTH-03, §11, §44) không định nghĩa organization hay workspace: user đăng nhập rồi tạo project trực tiếp, và project là ranh giới cách ly dữ liệu. Tenant mà Popper yêu cầu (architecture §7) chính là Project.
- Thêm `User owns Project` (1 — 0..n): người tạo project là owner (PRD FR-PROJ-01).
- Thêm `Project billed through BillingAccount` (1 — 0..1) thay cho liên kết qua Organization, vì usage/cost được theo dõi theo project (PRD §37).
- Khung Platform của sơ đồ: User, Role, ProjectMember, Project, BillingAccount, Dataset/DatasetSnapshot, Comment, Notification, PublicationRelease.
- File `.drawio` được vá trực tiếp (không chạy lại builder) để giữ các chỉnh tay trước đó; `build_conceptual_erd.py --check` vẫn pass trên model mới.
- Không sửa BRD/PRD: thay đổi này đưa ERD về đúng với hai tài liệu đó.

## 2026-10-01 — Điều chỉnh BRD/PRD theo kiến trúc Popper cập nhật

### Changed

- Cập nhật [BRD v1.10](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md) và [PRD v1.9](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md), giữ nguyên tên file, cấu trúc module, thứ tự và toàn bộ mã requirement; không thêm module/BR/FR mới.
- Nguồn đối chiếu là [Popper target architecture](../popper/docs/architecture.md) hiện tại: problem-first brief, Understand ⇄ Ground ⇄ Discover ⇄ Verify ⇄ Communicate, coordinator với bounded playbook; worker/decision layer là tùy chọn được đánh giá.
- Sửa các câu mâu thuẫn trực tiếp: agent được viết/chạy/debug code trong sandbox; Verify tùy chọn; reserve unread units không cần approval, pin/freeze cần recorded approval; một primary look đã bắt đầu không được rerun để thay outcome.
- BR-79 / Module AI: passive durable exposure/recording, round-scoped contract, Lineage/Program Error Plan kế thừa qua run, không cấp α mới cho mỗi batch/run. BR-60/80/81/87 và Module AF/AJ: tiếp tục nghiên cứu trên non-reserved data sau Verify, giữ origin/history, feedback/steering và recovery.
- Đồng bộ summary, rules, acceptance criteria, provenance, risk và demo bị ảnh hưởng. Giữ các giá trị đề xuất của defaults, làm rõ one-batch demo grant và caller cap; run không Verify vẫn cung cấp exploratory figures, interpretation, code, sources và history.
- Giữ [architecture.md](architecture.md) trong repo này như bản tham chiếu cũ; link kiến trúc hiện hành trong BRD/PRD trỏ trực tiếp tới repo `popper` bên cạnh, cần giữ cấu trúc thư mục này để resolve. Không sửa diagram/builder trong phạm vi BRD/PRD được yêu cầu; diagram hiện có cần đồng bộ ở đợt riêng trước commit/PR theo [AGENTS.md](AGENTS.md). Không commit/push/PR trong đợt này.

### Validation

- Đối chiếu source architecture §1, §3.4–§3.6, §6–§10; [pipeline §2, §7.2](../popper/docs/subsystems/pipeline.md), [error control §2](../popper/docs/subsystems/error-control.md), [runtime](../popper/docs/subsystems/runtime.md).
- Kiểm tra 150 mã requirement trong BRD và 197 trong PRD: giữ nguyên danh sách/thứ tự so với HEAD; local links và code fences — pass.
- Rà các rule hiện hành về generated code, reserved access, Verify retry và lineage error scope; kiểm tra cross-reference ID và link trong entry này — pass.
- `git diff HEAD --check` — pass. Không chạy generator/render check vì không sửa diagram.
- Các link hỏng trong entry changelog cũ được giữ ngoài phạm vi; entry mới không thêm link hỏng.

## 2026-09-30 — Thêm Conceptual ERD

### Added

- Thêm [Conceptual ERD](diagrams/erd/conceptual-erd/AI-Research-Experimentation-Platform-Conceptual-ERD.drawio) dạng `.drawio` editable: 43 entity, 70 relationship, ký hiệu crow's foot, connector vuông góc.
- Sơ đồ chia hai khung:
  - Platform (calling service): Organization, User, Role, ProjectMember, Project, BillingAccount, Dataset/DatasetSnapshot, Comment, Notification, PublicationRelease.
  - Popper (AI Scientist runtime), khung nét đứt. Các đường nối cắt qua ranh giới này là interface platform ↔ Popper (Popper architecture §4).
  - Các record của Popper mà platform dùng tới được tô riêng: PolicyVersion, DecisionRequest, UsageRecord, EvaluationDecision, DatasetAssessment.
- Thêm [ERD model](diagrams/erd/conceptual-erd/conceptual_erd.json) và [ERD builder](diagrams/erd/conceptual-erd/build_conceptual_erd.py). Model ghi reference nguồn cho từng entity. Builder có `--check` để kiểm tra geometry của connector, và hỗ trợ khung nhóm.
- Nguồn: `docs/` của repo `popper`, gồm architecture §1, §4, §6, §7; state §1–§2; research-program §1–§2; error-control §2; Stage 1–5; runtime §7–§10; views-and-visualization §1; evaluation §5.
- Ở mức conceptual, sơ đồ bỏ Artifact envelope, events, traces và các entity phụ (Construct, DataLead, Critique, Theory, Campaign, MethodLesson, Statement). Các phần này sẽ nằm trong logical ERD.
- File `.drawio` đã được chỉnh tay trong draw.io sau khi build: bỏ legend, dòng Source, scope note và các note sends/receives; rút gọn tiêu đề hai khung. Builder chưa tái tạo các chỉnh sửa này, nên chạy lại builder sẽ ghi đè chúng.
- `.gitignore`: bỏ qua output build, file tạm Office, `.DS_Store`, `__pycache__/`. Thêm `.vscode/settings.json` (port Live Server).
- Không sửa BRD/PRD/Use Case Spec vì ERD chỉ mô hình hoá lại dữ liệu đã có trong tài liệu, không đổi requirement.

### Validation

- `build_conceptual_erd.py --check`: pass, 0 error; còn 24 điểm cắt, đều hiển thị bằng jump arc.
- Render check file `.drawio` hiện tại bằng draw.io viewer (`viewer-static.min.js`) trên Chrome headless: pass.
- `git diff --check`: pass.

### References

- [architecture.md](architecture.md)
- Repo `popper`: `docs/architecture.md`, `docs/subsystems/*.md`

## 2026-09-28 — Đồng bộ BRD v1.9 / PRD v1.8 với kiến trúc đích Popper

### Changed

- Cập nhật [BRD](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md) lên v1.9 và [PRD](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md) lên v1.8 theo mục "Proposed BRD/PRD changes" và bảng mapping §7.10 của [architecture.md](architecture.md). Tên file giữ nguyên để không làm hỏng link và builder hiện có.
- Research loop: thay luồng tuyến tính single-path bằng agent loop trong giới hạn budget và invariant, giữ state machine cũ làm workflow floor (BR-80, PRD Module AJ, Section 38).
- Thêm exploration/confirmation split, Analysis Ledger, look budget, Research Protocol và confirmation batch (BR-79, PRD Module AI).
- Evidence Sufficiency Gate trở thành deterministic: Finding / Negative Result / Inconclusive tính trên adjusted interval so với δ_F / δ_N (BR-59, PRD Module AE). `NEED_MORE_EVIDENCE`, `TRY_ALTERNATIVE_METHOD` và `REPLICATE` chuyển thành next move (BR-60, Module AF); `NEED_HUMAN_REVIEW` chuyển thành escalation.
- Escalation dùng deterministic trigger làm mức sàn; confidence của decision layer chỉ được thêm escalation (BR-61, BRule-28).
- BR-78: các candidate ngang điểm cùng vào một confirmation batch trong look budget, không mở nhánh song song; bỏ `co_selected_with` và các cap cho nhánh song song, thêm quan hệ `registered_in`.
- Bỏ `run_python`/`run_sql`: experiment chỉ chạy qua registered capabilities (BR-22/23, PRD Module L).
- BR-21 được đổi thành triangulation / bounded robustness. BR-65 được đổi thành Prior-Work Assessment giới hạn theo coverage, không có nhãn `novel`. BR-48 dùng origin do hệ thống gán. Thêm BR-81 cho Research Program branches (Should).
- [architecture.md](architecture.md): sửa link BRD/PRD đang trỏ tới file `-REQ-TOOL.md` không tồn tại thành link tương đối tới file DRAFT; đổi đoạn "Proposed BRD/PRD changes" thành đã áp dụng, vẫn chờ supervisor sign-off.
- **Chưa đồng bộ diagram:** các builder Activity/Sequence/Use Case/Context vẫn mô tả outcome cũ (`ENOUGH_EVIDENCE`…) và luồng tuyến tính, nên cần cập nhật và regenerate trong một đợt riêng trước khi commit theo [AGENTS.md](AGENTS.md).
- Use Case Specification không có file canonical trong inventory nên không có gì để sửa.
- Lượt verify sau đó sửa thêm những chỗ còn mâu thuẫn với kiến trúc:
  - BR-07 / FR-PROFILE-04: quan hệ giữa hai biến chỉ được tính trên exploration partition.
  - BR-12 / FR-CLEAN-04: cleaning sau khi split là protocol deviation.
  - BR-41 / Module W: report hiển thị Negative Result và Inconclusive ngang Finding.
  - BR-42: checklist có thêm confirmation batch, severity checks và claim level.
  - BR-59 / Module AE: ghi rõ trường hợp two-sided và directional.
  - BRule-27 / PP-11: bỏ "re-plan" khỏi technical retry.
  - FR-EVAL-04: danh sách ablation khớp với BR-46.
  - FR-VIZ-02: figure chỉ được vẽ bằng registered chart capability.
- Lượt kiểm tra thứ ba:
  - Chuyển BR-78 từ section 11 (Business Rules) về section 10 (Business Requirements). Lỗi đặt sai vị trí này có từ v1.7.
  - BR-25: tách stability check trên cùng dữ liệu khỏi replication trên dữ liệu mới.
  - BR-16 / FR-RQ-01/02: thêm các trường của Research Brief, gap và intake (proceed / restate / clarify) trước khi đọc dữ liệu.
  - BR-44/45 / FR-EVAL-02 và Benchmark Task Structure: thêm null / planted-signal ground truth, false-finding rate và power.
  - FR-ADMIN-04: thêm run / policy / evaluation settings có policy version.
- Audit coverage toàn bộ architecture.md, đối chiếu từng section §4–§25 với BRD/PRD:
  - BRD thêm các BR mới:
    - BR-82 Research Knowledge Layer
    - BR-83 Research Artifacts & Visibility (Must)
    - BR-84 Publication Views & Integrity Audit (Must)
    - BR-85 Simulation Lab
    - BR-86 Theory & Observable Implications
    - BR-87 Research Campaign
  - BRD thêm BRule-39 (policy settings có version, không được làm yếu bảo đảm), GA-17/18, R-33→36 và acceptance criteria 59–63.
  - BRD mở rộng các BR đang có:
    - BR-27: yêu cầu cho từng claim type / evidential status
    - BR-34: reverse provenance
    - BR-40: limitation theo validity type
    - BR-49: dataset-scope looks
    - BR-55: replay vs re-derivation, tombstone
    - BR-56: tách các loại state
    - BR-58: screens, admission gate, circuit breaker
    - BR-63: champion/challenger
    - BR-80: role profiles, context
    - BRule-07: causal endorsement
    - NFR-02/07
  - PRD thêm Module AK (Artifacts & Publication Views), Module AL (Knowledge Layer), Module AM (Simulation Lab).
  - PRD thêm các FR: FR-STATE-06, FR-REFLOOP-06, FR-LOOP-09, FR-VALID-05/06, FR-PROV-04, FR-REPRO-04/05, FR-EVAL-06, FR-RQ-04.
  - PRD mở rộng NFR-02/03, acceptance criteria 61–66, entities, release plan và traceability.
- Thêm [PRD §53.1 Proposed Defaults](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md): bảng giá trị đề xuất cho các open decision 5–7 và 14–24, gồm α = 0.05, deciding interval Bonferroni 1 − α/L (Holm chỉ để báo cáo, FDR chỉ cho exploration), margin theo từng summary measure, bảng cỡ mẫu, δ_F/δ_N cho pack `general` và software engineering, split 50/50, look budget 5, mode và threshold của từng decision point ở A1, margin evaluation và danh sách capability MVP. BRD KPI-28 giờ trỏ tới α đề xuất. Tất cả đều đang chờ nhóm và supervisor duyệt.
- Kiểm tra lại phần định lượng của §53.1 (tính lại bằng power calculation):
  - Sửa Holm thành simultaneous interval Bonferroni cho quyết định sufficiency, vì Holm không cho ra simultaneous CI. BRD BR-49 và PRD FR-MTEST-02 được làm rõ theo.
  - Sửa số run evaluation từ 200 thành khoảng 655 mỗi arm cho margin 0.03; với margin 0.02 thì cần khoảng 1.470.
  - Thêm margin theo từng summary measure (d, r, β, OR, Cramér's V, η²) và bảng hướng dẫn cỡ mẫu.
- Kiểm tra tự động sau audit — pass:
  - BR/BRule đúng thứ tự và đúng section;
  - MoSCoW và traceability đầy đủ;
  - không có tham chiếu BR/BRule/BO/BP/GA/KPI/R/NFR/FR/Module tới ID không tồn tại;
  - không có FR trùng;
  - acceptance criteria PRD liên tục 1–66;
  - code fence cân bằng;
  - `git diff --check` sạch.

### Validation

- `git diff --check` — pass.
- Kiểm tra local link trong BRD, PRD, architecture.md và entry này — pass. Riêng [architecture.md](architecture.md) còn tham chiếu `research-methodology.md` và `roadmap.md`, hai file chưa có trong repository; lỗi này có sẵn từ trước và nằm ngoài phạm vi lần sửa này. Entry ngày 2026-09-24 bên dưới cũng đang link tới hai file `-REQ-TOOL.md`, vốn đã bị xoá khỏi working tree trước khi task này bắt đầu.
- Chưa chạy diagram generator/validator vì chưa sửa diagram.

### References

- [Target architecture](architecture.md)
- [BRD v1.9 Draft](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md)
- [PRD v1.8 Draft](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md)

## 2026-09-24 — Chuẩn hóa tài liệu REQ-TOOL và cập nhật package diagram

### Changed

- Thêm bản chuẩn hóa BRD v1.8 và PRD v1.7 theo định dạng REQ-TOOL; giữ nguyên hai file DRAFT nguồn.
- Cập nhật Use Case diagram editable cùng Activity và Sequence artifacts/builders theo các entry đồng bộ BRD/PRD ở dưới; trạng thái cuối có thay đổi Use Case diagram.
- Backup Context draw.io là artifact kỹ thuật, không phải nguồn nội dung.
- Giữ Context model và builder là nguồn cho Context DFD Level 0.

### Validation

- Chưa chạy validation diagram trong lượt cập nhật changelog này; kết quả validation của từng diagram được ghi tại entry tương ứng bên dưới.
- `git diff --check` — pass.

### References

- [BRD v1.8 Draft](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md) và [bản REQ-TOOL](AI-Research-Experimentation-Platform-BRD-v1.8-REQ-TOOL.md)
- [PRD v1.7 Draft](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md) và [bản REQ-TOOL](AI-Research-Experimentation-Platform-PRD-v1.7-REQ-TOOL.md)
- [Context — DFD Level 0](diagrams/context-diagrams/AI-Research-Experimentation-Platform-Context.drawio)
- [Activity — Main Flows](diagrams/activity-diagrams/AI-Research-Experimentation-Platform-Activity-Main-Flows.drawio)
- [Sequence — Main Flows](diagrams/sequence-diagrams/AI-Research-Experimentation-Platform-Main-Flows-Sequence.drawio)
- [Use Case — Platform](diagrams/usecase-diagrams/AI-Research-Experimentation-Platform-Use-Case.drawio)

## 2026-09-22 — Hoàn thiện 30 Sequence UC và chỉnh Activity theo BRD/PRD mới

### Changed

- Cập nhật [Activity — Main Flows](diagrams/activity-diagrams/AI-Research-Experimentation-Platform-Activity-Main-Flows.drawio) và [Activity builder](diagrams/activity-diagrams/update_activity_diagram.py): sửa thứ tự Hypothesis Selection Gate, đưa novelty về đúng lane, chuyển method-cap sang Experiment Planner / AI Agent, giới hạn reviewer override ở Researcher, cập nhật ownership của Evaluation và thêm nhánh Approved / Rejected–Revise–Automated Review cho Final Outputs & Manuscript.
- Cập nhật [Sequence — Main Flows](diagrams/sequence-diagrams/AI-Research-Experimentation-Platform-Main-Flows-Sequence.drawio) và [Sequence builder](diagrams/sequence-diagrams/build_sequence_diagrams.py) từ 24 lên đủ 30 trang UC. UC-25–UC-30 là các capability group đang có trong Use Case package: Research Definition, Hypotheses, Experiments, Research Findings, Research Outputs và System Administration.
- Sửa UC-10 để novelty có outer `opt [Novelty feature enabled]` và nhánh `SOURCE / NOT ASSESSED`; sửa UC-11 để human review là optional, cho phép high-confidence auto-continue.
- Không sửa Use Case diagram đang có trong worktree; không thay đổi nội dung BRD/PRD vì các requirement đã là source input của lần đồng bộ này.

### Validation

- `python -B -m py_compile` cho hai builder — pass.
- Sequence builder — pass: sinh đúng 30 pages, UC-01..UC-30 liên tục và unique.
- `activitylint.py --strict` — pass: 0 error, 0 warning.
- Activity `validate.py --strict --score` — pass: 0 error, 0 warning, score 0.
- Sequence `validate.py --strict --score` — pass: 0 error, 0 warning, score 0.
- Sequence semantic QA — pass: 30 pages; 0 actor-to-non-boundary message, 0 actor return source lỗi, 0 fragment parent reference lỗi.
- Draw.io 31.4.5 render check — pass trên các trang Activity 3/8/10/11 và Sequence 1/10/11/21/25–30.
- `git diff --check` — pass.

### References

- [BRD v1.8 Draft](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md)
- [PRD v1.7 Draft](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md)

## 2026-09-23 — Hoàn thiện System Context theo DFD Level 0

### Added

- Thêm [Context — DFD Level 0](diagrams/context-diagrams/AI-Research-Experimentation-Platform-Context.drawio) dạng `.drawio` editable cho AI Research Experimentation Platform.
- Thêm [context model](diagrams/context-diagrams/context_diagram.json) và [context builder](diagrams/context-diagrams/build_context_diagram.py), mô tả sáu external entity, một process trung tâm và 25 labeled data flow cấp cao theo BRD/PRD.
- Sắp xếp lại layout theo reference: Researcher ở trên, Project Manager/System Administrator bên trái, provider bên phải và Reviewer bên dưới; nới corridor để 25 connector không chồng nhau.
- Đặt từng flow label sát đúng làn mũi tên, rút gọn nhãn dài và tăng cỡ chữ để đọc được ở bản render.
- Loại các module nội bộ và Secure Sandbox khỏi external boundary; giữ AI/LLM Provider và Structured Decision Provider ở mức abstraction, không khóa vào vendor cụ thể.
- Không chỉnh sửa Use Case, Activity hoặc Sequence trong task này; các thay đổi sẵn có khác trong worktree được giữ nguyên.

### Validation

- Context builder — pass; builder dùng deterministic DFD Level 0 layout từ cùng model nguồn, không phụ thuộc Graphviz.
- Context `validate.py --strict --score` — pass: 0 error, 0 warning, score 0.
- Draw.io 31.4.5 render check — pass; artifact đã render thành PNG draft và được kiểm tra trực quan.
- `git diff --check` — pass.

### References

- [BRD v1.8 Draft](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md)
- [PRD v1.7 Draft](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md)

## 2026-09-22 — Đồng bộ Activity và Sequence với BRD v1.8 / PRD v1.7

### Changed

- Đồng bộ [Activity — Main Flows](diagrams/activity-diagrams/AI-Research-Experimentation-Platform-Activity-Main-Flows.drawio) và [Activity builder](diagrams/activity-diagrams/update_activity_diagram.py) với BR-64/65/73-75/78: Structured Idea Record/Reflection, optional novelty assessment, bounded tied candidates, immutable branch snapshots, sibling join, shared testing family, provenance/evaluation metadata và Final Outputs & Manuscript page.
- Đồng bộ [Sequence — Main Flows](diagrams/sequence-diagrams/AI-Research-Experimentation-Platform-Main-Flows-Sequence.drawio) và [Sequence builder](diagrams/sequence-diagrams/build_sequence_diagrams.py) cho UC-10–UC-16, UC-18–UC-24; giữ 24 pages và không thêm tree-search flow.
- Giữ nguyên [Use Case — Platform](diagrams/usecase-diagrams/AI-Research-Experimentation-Platform-Use-Case.drawio) theo đúng scope yêu cầu; BRD/PRD không cần sửa vì đã là source input của lần đồng bộ này.

### Validation

- `activitylint.py --strict` — pass: 0 error, 0 warning.
- Activity `validate.py --strict --score` — pass: 0 error, 0 warning, score 0.
- Sequence `validate.py --strict --score` — pass: 0 error, 0 warning, score 0.
- Sequence semantic QA — pass: 24 pages; không có actor-to-non-boundary message, actor return source lỗi hoặc fragment parent reference lỗi.
- Draw.io 31.4.5 render check — pass: 11 Activity pages và 24 Sequence pages đã render thành công để kiểm tra layout.
- Use Case SHA256 giữ nguyên so với trước khi thực hiện task.
- `git diff --check` — pass.

### References

- [BRD v1.8 Draft](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md)
- [PRD v1.7 Draft](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md)

## 2026-09-22 — Refresh editable diagram package

### Changed

- Cập nhật các diagram editable Activity, Sequence và Use Case cùng các builder
  source-backed tương ứng.
- Thêm các backup kỹ thuật đang nằm trong phạm vi staged; loại bỏ hai sequence artifact superseded.
- Không thay đổi BRD, PRD hoặc Use Case Spec vì đợt này không thay đổi
  requirement nghiệp vụ.

### Validation

- `activitylint.py --strict` — pass: 0 error, 0 warning.
- Activity `validate.py --strict --score` — 0 error, 1 cảnh báo crossing,
  score 10.
- Main Sequence `validate.py --strict --score` — pass: 0 error, 0 warning,
  score 0.
- Use Case `validate.py --strict --score` — 0 error, 56 cảnh báo geometry của
  system boundary/nested layout, score 355; giữ nguyên theo phạm vi yêu cầu.
- `git diff --cached --check` — pass trước khi commit.

## 2026-09-22 — Local tooling cleanup

### Removed

- Xoá metadata và worktree local của các công cụ agent không còn dùng.
- Loại bỏ các rule/reference tương ứng khỏi `AGENTS.md` và registry này.

### Validation

- Không có tài liệu canonical hoặc diagram editable nào bị xoá.
- Local links và cross-references được kiểm tra lại sau khi cleanup.

## 2026-09-22 — Documentation governance

### Added

- Thêm [`AGENTS.md`](AGENTS.md) với documentation-first gate: agent phải cập
  nhật tài liệu và changelog trước khi bắt đầu commit hoặc PR.
- Thêm registry này để liên kết tập trung tới toàn bộ tài liệu, flow reference,
  diagram editable và source builder.

### Validation

- Kiểm tra toàn bộ local link trong `changelog.md` và `AGENTS.md` — pass.
- `git diff --check` — clean.

### Scope note

- Task này chỉ thêm documentation governance và registry; không regenerate hoặc
  chỉnh sửa nội dung diagram.
- Các thay đổi diagram/source đang được stage từ trước được giữ nguyên, không
  gộp thêm thay đổi không liên quan.

### Current documentation package

#### Tài liệu nguồn

- [Business Requirements Document — BRD v1.8 Draft](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md)
- [Product Requirements Document — PRD v1.7 Draft](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md)
- Use Case Specification không có file canonical trong inventory hiện tại; Use Case diagram được giữ nguyên theo scope task.

#### Flow và diagram reference

- [Context — DFD Level 0](diagrams/context-diagrams/AI-Research-Experimentation-Platform-Context.drawio)
- [Activity — Main Flows](diagrams/activity-diagrams/AI-Research-Experimentation-Platform-Activity-Main-Flows.drawio)
- [Use Case — Platform](diagrams/usecase-diagrams/AI-Research-Experimentation-Platform-Use-Case.drawio)
- [Sequence — Main Flows](diagrams/sequence-diagrams/AI-Research-Experimentation-Platform-Main-Flows-Sequence.drawio)

#### Source builder và project entry point

- [Context model](diagrams/context-diagrams/context_diagram.json)
- [Context diagram builder](diagrams/context-diagrams/build_context_diagram.py)
- [Use Case diagram builder](diagrams/usecase-diagrams/build_usecase_diagram.py)
- [Sequence diagram builder](diagrams/sequence-diagrams/build_sequence_diagrams.py)
- [Repository README](README.md)
- [Agent rules](AGENTS.md)

#### Auxiliary artifact

- [Use Case diagram backup](<diagrams/usecase-diagrams/.$AI-Research-Experimentation-Platform-Use-Case.drawio.bkp>) — bản backup kỹ thuật, không phải source of truth.

## Quy ước registry

- BRD, PRD và Use Case Spec là nguồn nội dung nghiệp vụ/chức năng.
- PNG là flow reference trực quan; các file `.drawio`/`.xml` là artifact
  editable cần được giữ nguyên khả năng chỉnh sửa.
- Các file builder là source-backed generator; khi flow hoặc quan hệ thay đổi,
  phải cập nhật builder và regenerate artifact tương ứng.
- Metadata nội bộ của Git và các file tạm của môi trường không thuộc
  documentation package canonical này.
