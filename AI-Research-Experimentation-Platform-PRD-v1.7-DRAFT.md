# Product Requirements Document (PRD)
## AI Research Experimentation Platform

**Document Type:** Product Requirements Document\
**Version:** 2.0 Draft (2026-10-23)\
**Status:** Viết lại theo hệ thống đã hiện thực (`logos` contract 3.3.0, `ai-research-platform-fe`). Thay thế PRD v1.x.\
**Liên quan:** [BRD](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md) · [Architecture](architecture.md) · [changelog](changelog.md)

---

# 1. Product Summary

Web app cho nhóm nghiên cứu: tạo project, đưa dữ liệu vào, bắt đầu một **run**; các engine AI lần lượt tạo data card/research brief (nếu có dataset), hypothesis có bằng chứng và experiment plan đã review; researcher (hoặc AI PI ở mode auto) duyệt từng bước. Mọi artifact, review, quyết định và chi phí được lưu và truy vết.

# 2. Product Goals

| ID | Mục tiêu |
|---|---|
| G-01 | Một run đi trọn pipeline mà không cần thao tác ngoài platform. |
| G-02 | Người dùng thấy agent đang làm gì theo thời gian thực. |
| G-03 | Mọi artifact có finding review, diff revision và gap rõ ràng. |
| G-04 | Không run nào vượt token budget đã đặt. |
| G-05 | Admin cấu hình model, key, settings không cần redeploy. |

# 3. Non-Goals

Chạy experiment; tính finding; viết paper; organization/multi-tenant; custom role; hard delete project; giao diện riêng của engine.

# 4. Users & Roles

| Role | Phạm vi | Màn hình chính |
|---|---|---|
| Platform Admin (`platform_admin`) | Toàn hệ thống | `/admin/*` |
| Project Manager (`project_manager`) | Project | Workspace project, Members, Settings |
| Researcher (`researcher`) | Project | Datasets, Connections, Documents, Research flow |
| Reviewer (`reviewer`) | Project, chỉ đọc + comment | `/reviews`, Research flow |

Mọi user đã đăng nhập đều tạo được project và trở thành Project Manager của nó.

## 4.1 Ma trận quyền

| Hành động | Admin | PM | Researcher | Reviewer |
|---|:-:|:-:|:-:|:-:|
| Liệt kê project | tất cả | của mình | của mình | của mình |
| Đọc project, member, dataset, run, kết quả, comment | ✓ | ✓ | ✓ | ✓ |
| Sửa, archive, restore, complete, reopen project | ✓ | ✓ | – | – |
| Mời, đổi role, xoá member; huỷ/gửi lại lời mời | ✓ | ✓ | – | – |
| Upload/đổi tên dataset; upload/xoá project file | ✓ | ✓ | ✓ | – |
| Tạo, test, browse, import data connection | ✓ | ✓ | ✓ | – |
| Bắt đầu/sync run; trả lời gate; accept/send-back | ✓ | ✓ | ✓ | – |
| Abandon run | ✓ | ✓ | – | – |
| Comment; sửa comment của mình | ✓ | ✓ | ✓ | ✓ |
| Xoá comment | bất kỳ | bất kỳ | của mình | của mình |
| Xem audit log project | ✓ | ✓ | 404 | 404 |
| Khoá user, cấp admin, audit toàn hệ thống, trang admin | ✓ | – | – | – |

Không phải member → `404`; role không đủ → `403 ROLE_REQUIRED`.

# 5. Product Principles

| ID | Nguyên tắc |
|---|---|
| PP-01 | Traceability trước nội dung: không evidence ref thì không publish. |
| PP-02 | Số do code tính. |
| PP-03 | Run kết thúc với kết quả tốt nhất và gap, không dừng hẳn vì giới hạn. |
| PP-04 | Câu hỏi của agent không chặn run; default được ghi và đánh dấu. |
| PP-05 | Rule trong prompt đi kèm validator trong code. |
| PP-06 | Người quyết ở gate và accept; PI chỉ quyết thay ở mode auto, và luôn escalate integrity. |
| PP-07 | Không lưu, log hay trả về secret. |

# 6. Information Architecture

```text
/                         landing · /privacy · /terms
/auth/login → /auth/verify → /onboarding
/projects                 danh sách project
/projects/{id}            tổng quan workspace
  ├── datasets            dataset + version  → /datasets/{datasetId}
  ├── connections         data connections
  ├── files               Documents (project files)
  ├── experiments         Research flow: tạo run, Run studio, gate, stage, PI, artifact, comment
  ├── members             (PM) thành viên, lời mời
  └── settings            (PM) project settings, integrations, notification preferences, budget
/reviews                  hàng chờ của reviewer
/admin                    overview
  ├── users · users/{id}
  ├── projects · projects/{id}
  ├── providers           model theo role stage / review
  ├── decision-providers  TypeSafe integration
  ├── settings            admin settings + engine settings
  ├── usage · logs (audit) · log-monitoring · system
```

# 7. Primary User Flow

```text
1. Đăng nhập bằng mã email → đặt tên (onboarding)
2. Tạo project (thành PM) → mời researcher/reviewer
3. (Tuỳ chọn) Upload CSV/XLSX hoặc import từ connection → dataset version
4. Research flow → Start run: topic + domains | dataset version; mode light/auto; token budget
5. (light) Duyệt charter của PI
6. Stage dataset (nếu có): gate data_card_review → gate question_choice → publish brief → Accept
7. Stage hypothesis: gate shortlist → gate hypotheses → publish → Accept
8. Stage experiment_design: gate resources_review → gate plan_review → publish plan → Accept
9. PI close report → run completed → member đọc, tải artifact, comment
```

Ở bất kỳ stage nào sau khi `COMPLETED`: **Accept**, **Send back** về stage trước (iteration mới), hoặc **Rerun**.

# 8. Feature Modules

## 8.A Authentication & Account

- `POST /auth/email/start` gửi mã 6 số và link `{APP_URL}/auth/verify#token=...`; trả lời giống nhau cho mọi email; tạo tài khoản nếu chưa có. Mã/link hết hạn sau 10 phút; dùng một cái huỷ cái kia; 10 lần sai khoá mã 1 giờ.
- Session cookie HttpOnly, 30 ngày tuyệt đối, 7 ngày idle; CSRF token cho mọi `POST/PATCH/PUT/DELETE`.
- `user.onboarded=false` cho tới khi đặt tên (`PATCH /auth/me`).
- Google sign-in khi đủ cấu hình; chỉ nhận Gmail/Workspace đã verify.
- Rate limit: `email/start` 10/phút, các bước verify 20/phút → `429 RATE_LIMITED`.
- Admin tạo user, khoá (`USER_SUSPENDED`), cấp/thu `platform_admin`.

**AC:** đăng nhập lại sau reload qua `GET /auth/me` + `GET /auth/csrf-token`; request ghi thiếu CSRF → `403 CSRF_INVALID`.

## 8.B Project Workspace

- Trường: tên, mô tả, topic, domains, settings. Trạng thái suy ra: `draft` (chưa có dataset), `data_ready`, `researching` (có run active), `needs_review`, `completed` (PM đặt, reopen được), `archived`.
- Archive/restore; archived → chỉ đọc (`PROJECT_ARCHIVED`). Không hard delete.
- Members: mời qua email theo role, chấp nhận trong 24 giờ; gợi ý người mời qua `invite-candidates` (có SSE). Luôn còn một PM active (`LAST_PROJECT_MANAGER`).
- Documents: upload/tải/xoá project file.
- Activity feed và audit log project.

## 8.C Datasets

- Upload CSV hoặc XLSX (sheet đầu chuyển sang CSV); UTF-8, header unique, dòng đều, ≤ 50 MiB (`DATASET_MAX_UPLOAD_BYTES`).
- Mỗi file mới là version mới; đổi tên dataset; xem chi tiết và preview.
- Version được gửi cho engine qua `platform.dataset/v1`.

## 8.D Data Connections

- Connector: PostgreSQL, MySQL, BigQuery, Google Sheets, Google Drive (Google Picker), Prometheus, InfluxDB.
- Lưu, test, browse, preview, import bảng hoặc `SELECT` thành dataset version (timeout 300 giây).
- Credential mã hoá bằng `CONNECTION_SECRET_KEY`, không bao giờ trả về; thiếu key → `503 CONNECTIONS_NOT_CONFIGURED`. Chặn host loopback/private/link-local.

## 8.E Run & Research flow

**Tạo run** (`POST /projects/{id}/runs`):

| Trường | Ràng buộc |
|---|---|
| `topic` | 12–2000 ký tự; bắt buộc nếu không có dataset, nếu có thì thành objective |
| `dataset_version_id` | Tuỳ chọn; có thì dùng pipeline `dataset_first` |
| `domains` | Tối đa 6 |
| `review_mode` / `interaction_mode` | `light` (mặc định) hoặc `auto` |
| `token_budget` | > 0, ≤ `run_max_token_budget`; mặc định `run_default_token_budget` |

**Trạng thái run:** `queued`, `running`, `paused`, `awaiting_review`, `completed`, `budget_exceeded`, `failed`, `stopped`.

**Hành động:** `gates/{gate}` (trả lời gate), `pause`, `resume`, `stop`, `budget` (đặt tổng mới để resume), `sync`, `abandon` (PM; đánh dấu `failed`, không báo engine). Stage: `accept`, `send-back`, `rerun`, `reviews`.

**Recovery banner:** hiển thị `recovery.state` (`healing` / `needs_you`), lý do, lần thử, `next_try_at` và hành động gợi ý.

**Run studio:** đọc `GET .../events` + `events/stream` (SSE snapshot), dựng scene theo `step_key`: Scope, Search, Screen, Read, Synthesize, Hypothesize (debate), Map, Data, Experiment, Revision. Có hàng đợi trình chiếu và trạng thái "đang làm" khi engine im lặng. Hiển thị phase, step, gate mở, token đã dùng.

**AC:** tạo run thứ hai khi đang có run active → `409 RUN_ACTIVE`; engine không cấu hình → `503`; engine không truy cập được → `502`; contract khác major → `ENGINE_CONTRACT_MISMATCH`.

## 8.F Stage `dataset`

Step: `ingest → profile → describe → explore (tắt) → propose → brief`.

| Gate (light) | Kind | Researcher |
|---|---|---|
| `data_card_review` | artifact_review | Duyệt data card, sửa nhãn/khái niệm/role của biến |
| `question_choice` | question | Chọn research question đề xuất |

Output: `data_card.json` (`dataset.data_card/v1`), `data_card.md`, `brief.json` (`dataset.brief/v1`, primary), `brief.md`. Định dạng đọc được: csv, tsv, xlsx, sav, zsav, dta, parquet, zip. Settings: `min_cell` 5, `high_missing` 0.5, `max_levels` 30, `exploration_fraction` 0.3, `explore` false.

## 8.G Stage `hypothesis`

Step: `understand → search → screen → extract → synthesize → hypothesize → critique → verify → lock`.

| Gate (light) | Sau | Researcher |
|---|---|---|
| `shortlist` | screen | Approve; bỏ paper (`params.dropped`); reject → search lại |
| `hypotheses` | verify | Approve (`dropped`, `kept`); comment kèm finding → critique lại |

Output: `hypothesis.hypotheses/v1` (primary), `hypothesis.evidence/v1`, `hypothesis.argument_map/v1`; kèm novelty report, semantic graph, research canvas. Revise tự động khi review có blocking (≤ 2); vòng do researcher gửi lại ≤ 3. Advisory về chất lượng phát dưới dạng activity `hypothesis.advisory`.

**AC:** mỗi hypothesis có falsification criterion; mọi card/gap/hypothesis tham chiếu id tồn tại; không literature → lỗi `NO_LITERATURE`.

## 8.H Stage `experiment_design`

Step: `understand → ground → design → critique → verify → lock`.

| Gate (light) | Sau | Researcher |
|---|---|---|
| `resources_review` | ground | Duyệt resource; comment → ground lại |
| `plan_review` | verify | Duyệt plan và finding mở; comment → thêm một vòng critique |

Input: `hypothesis.hypotheses/v1` (bắt buộc), `hypothesis.evidence/v1`, `review.findings/v1` (tuỳ chọn). Output: `experiment_design.exp_plan/v1` (`exp_plan.md`, primary, byte đã review), `experiment_design.resources/v1`, `experiment_design.experiment_index/v1`. Status, approval, gap, assumption ở metadata. Settings: `stage_max_tokens` 2.000.000, `jev_max_calls` 24, `task_max_turns` 12, `task_max_tokens` 120.000, `resource_scout` false.

**AC:** plan chưa review xong vẫn publish với cờ không hoàn tất và finding mở; chỉ `no_plan` là thất bại.

## 8.I Review & Revision

- Mỗi vòng review công bố `review.findings/v1` (severity `blocking|major|minor`, category `leakage|baseline|control|statistics|reproducibility|evidence|other`); revision công bố `revision.diff/v1` và `revision.response/v1`.
- UI hiển thị các vòng review theo stage (`/stages/{id}/reviews`), diff giữa artifact (`/artifacts/{id}/diff`) và tải artifact.
- Send-back mở iteration mới; Platform ghi `iteration.summary` so sánh với iteration trước.

## 8.J AI Principal Investigator

- Tab PI trong run: charter, consultations, notebook.
- Researcher sửa charter (`PUT /pi/charter`), thêm directive (`POST /pi/directives`), endorse hoặc retry consultation.
- Ở `light`: PI khuyến nghị charter/outcome/failure, researcher quyết. Ở `auto`: PI quyết, Platform áp dụng; escalation `integrity`, `charter_conflict`, `cannot_decide` dừng lại chờ researcher.
- Critic (`pi_critic_enabled`, mặc định tắt) review mỗi stage hoàn tất: ≤ 8 point, trạng thái từng obligation.
- Close report nêu mọi obligation mở.

## 8.K Budget & Usage

- Token budget run; hết → `budget_exceeded`, tăng budget để resume; event `budget.updated`.
- Budget tháng của project (soft cap, cảnh báo); `GET /projects/{id}/spending/usage`.
- Admin `/admin/usage` theo thời gian, project, model.

## 8.L Comments, Notifications, Integrations

- Comment trên run: tạo, sửa của mình, xoá theo quyền.
- Thông báo trong app + SSE (`/notifications/stream`), đánh dấu đã đọc; preference theo project.
- Integration project (≤ 10): Slack, Microsoft Teams, Discord incoming webhook; event `run.awaiting_review`, `run.finished`; test gửi và lịch sử delivery.

## 8.M Administration

| Trang | Nội dung |
|---|---|
| Overview | Số liệu tổng quan hệ thống |
| Users | Danh sách, tạo, khoá, cấp admin |
| Projects | Xem mọi project |
| Providers | Model + key theo role `stage` / `review`; test kết nối |
| Decision providers | Integration TypeSafe |
| Settings | `pi_critic_enabled`, `max_auto_revisions` (2), `max_iterations` (3), `run_default_token_budget` (4M), `run_max_token_budget` (20M), `oversight_llm_timeout_seconds` (180), `oversight_contact_email`; settings từng engine |
| Usage | Token theo thời gian/project/model |
| Logs | Audit log toàn hệ thống |
| Log monitoring | Request event đã redact, error rate, cảnh báo, SSE |

# 9. Non-Functional Requirements

| ID | Yêu cầu |
|---|---|
| NFR-01 | Lỗi dự đoán được trả `APIError` có `code` ổn định và message tiếng Anh dễ hiểu; không trả 500 trần. |
| NFR-02 | SSE gửi snapshot, keep-alive 15 giây, `Cache-Control: no-cache, no-transform`; client reconnect nhận snapshot mới. |
| NFR-03 | Restart engine không mất run: run thành `paused` (`interrupted`), resume kèm key. |
| NFR-04 | Provider outage → retry với backoff trong 60 giây rồi pause, không fail. |
| NFR-05 | Secret mã hoá trong DB; key model chỉ trong bộ nhớ của engine/PI. |
| NFR-06 | Graceful shutdown 30 giây cho Uvicorn. |
| NFR-07 | Mọi thay đổi FE nhìn thấy được ghi trong `logos/docs/fe-contract-changes.md`. |

# 10. Release Status

| Hạng mục | Trạng thái |
|---|---|
| Auth, project, member, dataset, connection, files | Đã có |
| Pipeline `default` (hypothesis → experiment_design) | Đã có; tạm dừng phát triển trong lúc làm nhánh dataset |
| Pipeline `dataset_first` | Đang phát triển |
| AI PI | Đã có; Critic tắt mặc định |
| Run studio | Đã có cho stage hypothesis; các scene khác đang hoàn thiện |
| `explore` (Data Formulator) | Thử nghiệm, tắt |
| Experiment execution, paper | Ngoài phạm vi |
