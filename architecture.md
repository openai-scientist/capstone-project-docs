# Logos System Architecture

**Version:** 2.0 (2026-10-23)\
**Status:** Viết lại theo code hiện hành của monorepo `logos` (contract `3.3.0`) và frontend `ai-research-platform-fe`.\
**Thay thế:** "Popper Target Architecture" (v1.x). Kiến trúc Popper (Understand ⇄ Ground ⇄ Discover ⇄ Verify ⇄ Communicate, Analysis Ledger, Simulation Lab…) **không** được hiện thực và không còn là kiến trúc mục tiêu.

Liên quan: [BRD](AI-Research-Experimentation-Platform-BRD-v1.8-DRAFT.md) · [PRD](AI-Research-Experimentation-Platform-PRD-v1.7-DRAFT.md) · [changelog](changelog.md)

---

## 1. Mục đích

Tài liệu mô tả kiến trúc **đang chạy** của AI Research Experimentation Platform: các thành phần, trách nhiệm, ranh giới, luồng run, contract giữa Platform và engine, và các quy tắc cứng mà code thực thi. Nguồn sự thật là code; khi tài liệu và code lệch nhau, code thắng và tài liệu phải được sửa.

Phạm vi sản phẩm hiện tại: từ **topic / dataset** → **hypothesis có bằng chứng** → **experiment plan đã được review**. Hệ thống **không chạy experiment**, không viết paper, không tính kết quả thống kê trên dữ liệu thật để kết luận.

## 2. Tổng quan thành phần

```text
                  Browser
                     │  HTTPS, cookie session + CSRF, SSE
                     ▼
  ┌──────────────────────────────────┐
  │ Frontend (Next.js 16, React 19)  │  ai-research-platform-fe
  │ rewrite /api/v1 → Platform       │
  └──────────────────────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────┐        ┌─────────────────────────┐
  │ Platform  (services/management)  │──HTTP─▶│ oversight (AI PI)       │
  │ FastAPI + PostgreSQL + FileStore │◀─cb────│ PI agent + Critic agent │
  │ auth, projects, data, runs,      │        └─────────────────────────┘
  │ gates, budget, BYOK, audit       │
  └──────────────────────────────────┘
        │ engine contract (X-Service-Key)          ▲ callbacks: events, artifacts
        ▼                                          │
  ┌───────────┐   ┌──────────────┐   ┌───────────────────────┐
  │ dataset   │   │ hypothesis   │   │ experiment_design     │   stages/*
  │ engine    │   │ engine       │   │ engine                │   (logos_engine skeleton)
  └───────────┘   └──────────────┘   └───────────────────────┘
        │                 │                     │
        ▼                 ▼                     ▼
   LLM gateway (OpenAI-compatible, key do Platform gửi theo run)
   OpenAlex / literature providers, HuggingFace Hub, GitHub, PyPI (tuỳ engine)
```

| Thành phần | Thư mục | Import | Vai trò |
|---|---|---|---|
| Frontend | `ai-research-platform-fe` | — | Giao diện researcher/manager/reviewer/admin; Run studio hiển thị tiến trình engine |
| Platform | `logos/services/management` | `management` | Bản ghi của run: trạng thái, gate, quyết định, budget token, model key (BYOK), mọi hành động của người dùng |
| AI PI | `logos/services/oversight` | `logos_oversight` | Ra quyết định/khuyến nghị ở cấp pipeline (charter, outcome của stage, failure, close); Critic review stage (tắt mặc định) |
| Engine `dataset` | `logos/stages/dataset` | `logos_dataset` | Đọc bảng dữ liệu, xuất data card + research brief |
| Engine `hypothesis` | `logos/stages/hypothesis` | `logos_hypothesis` | Topic → literature → evidence card → hypothesis có falsification criterion |
| Engine `experiment_design` | `logos/stages/experiment-design` | `logos_experiment_design` | Hypothesis → experiment plan được review độc lập |
| Contract | `logos/packages/contract` | `logos_contract` | Wire model, route, event type, schema id; `CONTRACT_VERSION` |
| Engine skeleton | `logos/packages/engine` | `logos_engine` | Run loop, run store, HTTP routes, outbox, gate, pause/cancel, recovery, LLM client, agent loop, `decide` |
| Eval | `logos/eval` | `logos_eval` | Bench có chấm điểm (hypothesis cases, seeded-flaw recall của critic) |

Quy tắc đặt tên: thư mục, distribution, import, `engine_key` = `stage_key`, prefix schema/event/env đều trùng nhau (`experiment_design.exp_plan/v1`, `EXPERIMENT_DESIGN_*`).

## 3. Ranh giới trách nhiệm

| Ai | Sở hữu | Không được |
|---|---|---|
| Platform | Run state, gate, quyết định accept/send-back, token budget, model key, quyền truy cập, audit | Đọc hay đánh giá nội dung khoa học của artifact |
| Engine | Khoa học của stage: step, prompt, pack, vòng draft → review → revise của chính nó | Có CLI hay giao diện riêng cho researcher; đọc model key từ environment |
| AI PI | Quyết định/khuyến nghị pipeline theo charter | Kết thúc run (không có action `stop`); trả lời gate của engine |
| Contract | Tên wire, route, version, artifact schema | Chứa logic nghiệp vụ |

Import rules (`.importlinter`): `packages/*` không import `stages/*`/`services/*`; stage không import stage khác; service và stage không import nhau — chỉ nói chuyện qua HTTP.

## 4. Platform (`services/management`)

### 4.1 Stack

Python 3.12, FastAPI, Uvicorn; PostgreSQL 16 qua SQLAlchemy async (`asyncpg`), Alembic sở hữu schema; file lưu ở thư mục local hoặc Cloudflare R2; email qua Resend; secret trong DB mã hoá bằng `SecretBox` (Fernet).

### 4.2 Feature và layer

Mỗi feature là một package `router / schemas / service / models`. Layer từ cao xuống thấp, chỉ import xuống:

`research | monitoring` > `datasets` > `connections` > `projects` > `integrations` > `identity | notifications` > `audit | access` > `core | db | infra`

| Feature | Nội dung |
|---|---|
| `identity` | Đăng nhập không mật khẩu (mã 6 số / magic link), Google sign-in, user, avatar, onboarding |
| `access` | Kiểm tra role theo project |
| `projects` | Project, member, lời mời, comment, project file, settings, spending, activity |
| `connections` | Data connection: PostgreSQL, MySQL, BigQuery, Google Sheets, Google Drive, Prometheus, InfluxDB |
| `datasets` | Dataset CSV/XLSX có version, import từ connection |
| `integrations` | Webhook Slack / Microsoft Teams / Discord cho event `run.awaiting_review`, `run.finished` |
| `notifications` | Thông báo + SSE stream, preference theo project |
| `research` | Run, stage run, gate, review, artifact, PI consultation, orchestrator, healing, cost ledger, model profile, admin settings |
| `monitoring` | Admin overview, usage, log monitoring, telemetry |
| `audit` | Audit log project và toàn hệ thống |

### 4.3 Public API

Tất cả dưới `/api/v1`, envelope `success / message / data / meta` hoặc `error.code`. Nhóm route chính:

| Nhóm | Route |
|---|---|
| Health | `/health/live`, `/health/ready` |
| Auth | `/auth/email/start`, `email/verify`, `email/link-info`, `email/verify-link`, `/auth/me`, `/auth/csrf-token`, `/auth/google/*` |
| Projects | `/projects`, `/{id}` (+ `archive`, `restore`, `complete`, `reopen`), `members`, `invite-candidates(/stream)`, `files`, `settings`, `spending/usage`, `activity` |
| Data | `/projects/{id}/datasets`, `/projects/{id}/connections`, `/connections/google/*` |
| Runs | `/projects/{id}/runs` — `GET/POST`, `/{run}`, `events`, `events/stream`, `gates/{gate}`, `pause`, `resume`, `stop`, `budget`, `sync`, `abandon` |
| Stages | `/runs/{run}/stages` — list, `/{stage}/gates/{gate}`, `reviews`, `accept`, `send-back`, `rerun` |
| PI | `/runs/{run}/pi`, `notebook`, `consultations`, `consultations/{id}/endorse`, `/retry`, `PUT charter`, `POST directives` |
| Artifacts | `/runs/{run}/artifacts`, `/{artifact}/diff`, `/{artifact}/download` |
| Collaboration | `/runs/{run}/comments`, `/notifications(/stream)`, `/invitations` |
| Admin | `/admin/settings`, `/admin/engines/{engine_key}/settings`, `/admin/models/{role}(/test)`, `/admin/integrations/{name}`, `/admin/usage`, `/admin/overview`, `/admin/log-monitoring(/stream)`, `/admin/audit-log`, `/users` |
| Internal | `/internal/engines/{engine_key}/runs/{stage_run_id}/events|artifacts`, `/internal/engines/{engine_key}/artifacts/{id}`, `/internal/engines/{engine_key}/datasets/{version_id}`, PI callback `/consultations/{id}/result` — chỉ `X-Service-Key` |

SSE stream gửi snapshot (thay thế, không append), keep-alive 15 giây, header `Cache-Control: no-cache, no-transform`.

### 4.4 Pipeline cấu hình

`ENGINES` (JSON: engine key → `{base_url, service_key, timeout_seconds}`) và `PIPELINES` (JSON: tên → danh sách `{stage_key, engine, loop_back_to}`) là biến môi trường. Hai pipeline đang dùng:

| Pipeline | Stage | Khi nào |
|---|---|---|
| `default` | `hypothesis` → `experiment_design` | Run có topic, không có dataset (nghiên cứu ML/SE benchmark) |
| `dataset_first` | `dataset` → `hypothesis` → `experiment_design` | Run gắn `dataset_version_id`; experiment_design viết statistical analysis plan (data role `analysis`) |

### 4.5 Run, stage run và trạng thái

- Một project chỉ có **một run active** (`409 RUN_ACTIVE`).
- Run status: `queued`, `running`, `paused`, `awaiting_review` (active) · `completed`, `budget_exceeded`, `failed`, `stopped` (không active).
- Stage run status: `PENDING`, `RUNNING`, `COMPLETED`, `ACCEPTED`, `FAILED`, `STOPPED`, `SUPERSEDED`.
- Run có `iteration`; send-back về stage trước mở iteration mới (giới hạn `max_iterations`, mặc định 3), stage cũ thành `SUPERSEDED`, Platform ghi `iteration.summary`.
- `review_mode` / `interaction_mode`: `light` (researcher trả lời gate) hoặc `auto` (chạy tự động, PI quyết định thay).
- Platform event (class `platform`): `stage.status_changed`, `decision.recorded`, `iteration.started`, `budget.updated`, `attention.required`… cùng luồng với event lifecycle/activity của engine, đánh số `seq`.

### 4.6 Self-healing và recovery

`healing.py`: lỗi hạ tầng không kết thúc run — stage lỗi tạm thời được sweep chạy lại với thời gian chờ tăng dần. Lỗi cần con người thì dừng stage và nói rõ. `RunItem.recovery` báo `state` (`healing`/`needs_you`/`none`), `reason`, `attempt` và `action` gợi ý (`wait`, `rerun`, `raise_budget`, `accept`, `answer_gate`, `redo`, `endorse_charter`, `answer_pi`). Platform không poll engine; `POST .../sync` hỏi trạng thái stage hiện tại.

### 4.7 Budget và chi phí

- **Token budget của run** (input + output): mặc định 4.000.000, tối đa 20.000.000 (admin setting). Mỗi stage nhận phần còn lại. Hết budget → engine tự pause lý do `budget`, run thành `budget_exceeded`; `POST .../budget` với tổng mới sẽ resume. PI consultation tính vào cùng budget.
- **Cost ledger**: engine báo tổng luỹ kế; ledger chỉ ghi phần tăng, có `source_key` để report lặp không bị tính hai lần.
- **Budget tháng của project** (`project_budget.py`): soft cap khi bắt đầu công việc, có cảnh báo; không reserve trước.

### 4.8 Model key (BYOK) và settings

Model key là admin setting trong DB, không phải env. Admin cấu hình model theo role `stage` và `review` (`/admin/models/{role}`, có `test`), cùng integration như TypeSafe (`/admin/integrations/{name}`). Key được gửi kèm từng request tới engine/PI và chỉ sống trong bộ nhớ ở phía đó.

Admin settings (`/admin/settings`): `pi_critic_enabled` (false), `max_auto_revisions` (2), `max_iterations` (3), `run_default_token_budget`, `run_max_token_budget`, `oversight_llm_timeout_seconds` (180), `oversight_contact_email`. Engine settings qua `/admin/engines/{engine_key}/settings` và được gửi theo mỗi run.

## 5. Engine contract (`packages/contract`, version 3.3.0)

### 5.1 Route của engine

| Method | Path | Ý nghĩa |
|---|---|---|
| GET | `/health` | Health (chi tiết chỉ khi có key hợp lệ) |
| GET | `/capabilities` | Step, gate, schema, settings model, activity types |
| POST | `/runs` | `RunCreateRequest` → `RunCreateResponse`; idempotent theo `stage_run_id` |
| GET | `/runs/{id}` | `RunState` |
| GET | `/runs/{id}/events?after&limit` | `EventPage` |
| POST | `/runs/{id}/gates/{gate}` | `GateAnswer` |
| POST | `/runs/{id}/pause|resume|cancel` | Điều khiển run |

Engine callback về Platform: `.../events` (`EventBatch`) và `.../artifacts` (multipart `meta` + `file` → `artifact_id`). Input artifact tải qua `download_url` bằng service key.

### 5.2 Hằng số

- Phase: `drafting`, `reviewing`, `revising`, `awaiting_researcher`, `published`, `failed`, `cancelled`, `paused` (terminal: `published`, `failed`, `cancelled`).
- Mode: `light`, `auto`. Model role: `stage`, `review`. Gate kind: `question`, `artifact_review`.
- Finding: severity `blocking|major|minor`; kind `fix|needs_evidence`; category `leakage|baseline|control|statistics|reproducibility|evidence|other`. Review pass khi không có finding `blocking`. Finding id `F<round>-<n>`.
- Lifecycle event: `run.phase_changed`, `run.step_changed`, `run.gate_opened`, `run.gate_closed`, `review.completed`, `artifact.delivered`, `run.published`, `run.failed`, `run.cancelled`, `run.cost`. Activity event có prefix `<engine_key>.`.
- `inputs.params.guidance`: tối đa 20 dòng × 500 ký tự (`Research question:`, `Objective:`, `Success criterion:`, `Directive:`, `Handoff:`), Platform tính lại ở mỗi lần bắt đầu stage.

### 5.3 Artifact schema

| Schema | Producer | Ghi chú |
|---|---|---|
| `platform.dataset/v1` | Platform | Input của `dataset` |
| `dataset.data_card/v1`, `dataset.data_card_markdown/v1` | dataset | Data card kiểu SPSS Variable View |
| `dataset.brief/v1` (primary) | dataset | Objective + một research question; ghi SHA-256 của data card |
| `hypothesis.hypotheses/v1` (primary), `hypothesis.evidence/v1`, `hypothesis.argument_map/v1` | hypothesis | |
| `experiment_design.exp_plan/v1` (primary), `experiment_design.resources/v1`, `experiment_design.experiment_index/v1` | experiment_design | Markdown plan đúng byte đã review |
| `review.findings/v1`, `revision.diff/v1`, `revision.response/v1` | mọi engine | Platform đọc để hiển thị review/revision |
| `pipeline.feedback/v1`, `pipeline.recommendation/v1` | Platform/PI | Phản hồi giữa stage |

Artifact bất biến; version là số nguyên dương riêng với major version trong schema id.

### 5.4 Vòng đời một stage

```text
POST /runs ─▶ drafting ─▶ reviewing (reviewer độc lập) ─┬─ pass ─────────────▶ awaiting_researcher (gate, light)
                 ▲                                      └─ blocking ─▶ revising ┘          │
                 └──────────── comment của researcher ◀───────────────────────────────────┤
                                                                                           ▼
                                                                    published (primary artifact)
                                                                                           │
                    Platform: stage COMPLETED ─▶ accept (ACCEPTED, sang stage sau) | send-back (iteration mới) | rerun
```

Approve gate và accept stage là hai hành động riêng. Send-back chỉ từ stage `COMPLETED`.

## 6. Engine skeleton (`packages/engine`)

Một engine chỉ khai báo `EngineSpec` trong `engine.py` và viết các step; skeleton lo phần còn lại.

- **Spec**: `steps=[Step(key, label, phase, run)]`, `edges` + `max_loops` cho mọi back edge, `gates=[GateSpec(...)]` (chỉ mode `light`, tối đa 2 gate), `output_schemas`, `primary_artifact`, `input_schemas`, `settings_model`, `activity_types`.
- **Context**: `ctx.llm(role)`, `ctx.read_*/write`, `ctx.input(schema)`, `ctx.params`, `ctx.emit`, `ctx.assume`, `ctx.checkpoint`, `ctx.usage`, `ctx.upload`, `ctx.publish_review`, `ctx.publish_revision`.
- **Run store**: thư mục `<RUNS_ROOT>/<engine_run_id>/`; step được bỏ qua khi resume nếu manifest khớp hash output và input upstream; back edge chuyển step bị ảnh hưởng vào `attempts/<n>/`.
- **LLM client** (`OpenAIClient`): retry lỗi tạm (408, 429, 5xx) với backoff, `ProviderUnavailable` khi outage kéo dài → run pause chứ không fail. Một chính sách usage chung: call không có usage bị tính trần (`max_tokens`) và phát `<engine>.usage.assumed`.
- **Agent loop** (`run_agent`): tool-calling có giới hạn turn/token, tool `finish` với check bằng code; lỗi tool trả về cho model, không làm fail run. Có tool `ask_user` tuỳ chọn.
- **`decide`**: chọn một trong các option do code liệt kê khi không có rule — TypeSafe (nếu có integration) → một LLM call role `review` → option `safe`. Không bao giờ chặn run.

Quy tắc skeleton thực thi:

- **R1 — Run kết thúc, không dừng hẳn**: giới hạn (vòng revise, token cap) kết thúc sớm với `review_complete=false` và ghi gap. Chỉ cancel hoặc lỗi kỹ thuật mới dừng hẳn.
- **R3 — Câu hỏi không chặn run**: `ctx.assume` ghi default và lý do; câu trả lời hệ thống được đánh dấu không phải ý định researcher.
- **Key chỉ trong bộ nhớ**: sau restart, run đang chạy thành `paused` lý do `interrupted`; Platform resume kèm key (`409 CREDENTIALS_REQUIRED` nếu thiếu).

## 7. Các engine

### 7.1 `dataset`

`ingest → profile → describe → explore → propose → brief`; gate `data_card_review` (artifact_review, sau `describe`) và `question_choice` (question, sau `propose`); back edge `describe→describe`, `propose→propose`.

- Đọc csv, tsv, xlsx, sav, zsav, dta, parquet, zip (kèm codebook). Mọi con số do code tính; model chỉ thấy data card, không bao giờ thấy dòng dữ liệu; không dòng dữ liệu nào bị copy vào artifact.
- Bảo vệ: chỉ liệt kê value khi ≤ `max_levels` (30); level < `min_cell` (5) bị gộp; cột text gần-unique là `free_text`; mỗi thuộc tính có `*_source`; chia `exploration_fraction` (0,3).
- `explore` (Data Formulator) **tắt mặc định** vì DF cho model thấy sample row và sandbox local không phải ranh giới bảo mật.

### 7.2 `hypothesis`

`understand → search → screen → extract → synthesize → hypothesize → critique → verify → lock` (khoa học stage 1–9 phỏng theo AutoResearchClaw, MIT).

| Gate | Sau | Researcher có thể |
|---|---|---|
| `shortlist` | `screen` | approve; bỏ paper (`params.dropped`); reject (chạy lại `search`) |
| `hypotheses` | `verify` | approve (`dropped`, `kept`); comment kèm finding (chạy lại `critique`) |

- Loop: `critique→critique` khi review có blocking (≤ `max_auto_revisions`, mặc định 2); `screen→search`, `verify→critique` theo câu trả lời researcher (≤ 3 vòng). Loop cạn thì tiếp tục và báo gap.
- Evidence rules: mỗi paper có provenance; mỗi card/gap/hypothesis tham chiếu record lưu trữ; card ở mức abstract (`evidence_scope: "abstract"`); paper không chấm điểm bị loại có lý do; novelty report là heuristic; hypothesis phải có falsification criterion cụ thể.
- Không fallback/fake output: thiếu model, nguồn literature hay credential → lỗi rõ (`NO_LITERATURE`, `EMPTY_SHORTLIST`…).

### 7.3 `experiment_design`

`understand → ground → design → critique → verify → lock`.

| Step | Việc |
|---|---|
| `understand` | Snapshot input nguyên văn, frame câu hỏi, suy ra research type, ghi assumption |
| `ground` | Dataset, benchmark, baseline, code công khai có nguồn kiểm chứng (tuỳ chọn resource scout) |
| `design` | Plan: protocol, measurement, analysis, seed, compute + experiment index |
| `critique` | Methodologist và Skeptic độc lập; routing REVISE / REFRAME (judge TypeSafe JEV khi cả hai hợp lệ) |
| `verify` | Final check mới với code check, diff, convergence |
| `lock` | Đóng băng byte của plan kèm hash và publish |

- Gate (light): `resources_review` sau `ground`, `plan_review` sau `verify`. Comment ở `plan_review` gửi plan qua `critique` thêm một vòng.
- Thống kê chỉ do calculator trong `science/registry.py` tính, mỗi calculator có test với giá trị tham chiếu độc lập. Không bịa evidence, tính toán, câu trả lời hay sự chấp nhận.
- Plan chưa review xong vẫn publish với gap và finding mở; publish **không** phải là researcher chấp thuận hay quyền chạy experiment. Thất bại duy nhất là `no_plan`.
- Mặc định: `stage_max_tokens` 2.000.000, 24 JEV call, 12 turn và 120.000 token mỗi specialist.

## 8. AI Principal Investigator (`services/oversight`)

Platform hỏi PI qua `POST /consultations` (`ConsultationRequest`: charter, situation, allowed actions, lab notebook, artifact refs, model + key); PI trả `202`, chạy agent nền và post `ConsultationResult` về callback.

| Kind | Ai quyết (`authority.py`) | Nội dung |
|---|---|---|
| `charter` | PI ở `auto`, researcher ở `light` | Research question, objective, success criterion |
| `stage_review` | Report (Critic) | Kiểm charter fit, obligation, directive, integrity; ≤ 8 point |
| `stage_outcome` | PI ở `auto`, researcher ở `light` | Accept / send-back sau khi stage hoàn tất |
| `failure` | PI ở `auto`, researcher ở `light` | Xử lý stage lỗi |
| `close` | PI | Báo cáo đóng run, nêu mọi obligation mở |

- Tool của PI: `list_artifacts`, `read_artifact`, `read_notebook`, `search_literature` (OpenAlex), `check_dataset` (HuggingFace Hub), `write_note`; kết thúc bằng `decide`. Critic chỉ có tool đọc và kết thúc bằng `report`.
- Guard trong code: action hợp lệ, id đã biết, không accept item bị chặn trừ khi nêu là known risk, trả lời mọi Critic point nghiêm trọng, trích mọi directive của researcher. Vi phạm quay lại agent như observation.
- PI không bao giờ kết thúc run; không trả lời gate của engine (`gate_*` → 422).
- Escalation `integrity`, `charter_conflict`, `cannot_decide` chuyển quyết định cho researcher kể cả ở `auto`.
- **Obligation**: rủi ro được chấp nhận ở một stage mà stage sau phải kiểm soát; Platform tạo khi accept vượt finding blocking/Critic point, `guidance` báo cho engine.
- Researcher tác động qua `PUT /pi/charter`, `POST /pi/directives`, `endorse`/`retry` consultation. PI không giữ run state; mỗi consultation lưu một file JSON đã redact.

## 9. Frontend (`ai-research-platform-fe`)

Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS v4, shadcn/ui, TanStack Query/Table, Redux Toolkit; Biome, Vitest.

| Khu vực | Route |
|---|---|
| Công khai | `/`, `/privacy`, `/terms` |
| Đăng nhập | `/auth/login`, `/auth/verify`, `/onboarding` |
| Project | `/projects`, `/projects/{id}`, section `datasets`, `connections`, `files` (Documents), `experiments` (Research flow / Run studio), `members`, `settings`; `/projects/{id}/datasets/{datasetId}` |
| Reviewer | `/reviews` |
| Admin | `/admin`, `users`, `projects`, `providers`, `decision-providers`, `settings`, `usage`, `logs`, `log-monitoring`, `system` |

**Run studio** (`_components/run-studio`) dựng scene theo `step_key` từ run event stream: Scope, Search, Screen, Read, Synthesize, Hypothesize (debate board), Map, Data, Experiment, Revision panel… FE có hàng đợi trình chiếu để làm mượt event đến theo đợt. Thư mục `dashboard/*` là template còn sót, không thuộc sản phẩm.

## 10. Bảo mật

- Session cookie HttpOnly (30 ngày tuyệt đối, 7 ngày idle), CSRF token cho mọi request ghi, CORS allowlist, rate limit theo IP cho auth.
- Không phải member → `404`; role không đủ → `403 ROLE_REQUIRED`.
- Credential của connection và model key mã hoá bằng `SecretBox`, không bao giờ trả về hay log. Connection từ chối host loopback/private/link-local.
- Service-to-service: `X-Service-Key` riêng cho từng engine và PI; callback không hợp lệ nhận `401 SERVICE_KEY_INVALID` trước khi đọc body.
- Engine và PI không đọc key từ environment.

## 11. Triển khai

- Dev: `logos/deploy/docker-compose.dev.yml` (build context là repo root, Postgres host port 55432).
- Production: `logos/deploy/docker-compose.prod.yml`, deploy bằng Dokploy từ image do workflow Deploy đẩy sau khi CI pass trên `main`. Service `migrate` chạy `alembic upgrade head` trước API.
- API bind `127.0.0.1:8080` sau reverse proxy TLS; proxy phải tắt buffering cho SSE.

## 12. Quy tắc cứng xuyên suốt

1. **Gate kiểm tra traceability, không kiểm tra đúng sai**: code chỉ từ chối provenance/contract hỏng (sai shape, id, evidence ref không resolve, số bịa đặt trình bày như có nguồn). Vấn đề nội dung khác được retry, rồi giữ thành finding hoặc known risk; không làm fail run.
2. **Số liệu trong plan/kết quả đến từ code** (calculator, runner), không từ text của LLM.
3. **Mỗi hard rule trong prompt đều có validator** trong code (và ngược lại).
4. **Không giữ hay log key**; không đọc `.env`.
5. **Thay đổi FE nhìn thấy được** (route, shape, status code, enum) phải ghi vào `logos/docs/fe-contract-changes.md`; thay đổi wire tăng `CONTRACT_VERSION` (additive → minor, breaking → major và cần quyết định của người dùng).

## 13. Kiểm chứng

| Lệnh | Khi nào |
|---|---|
| `task verify:fast P=<project>` | Mỗi thay đổi: import rules + ruff + test của project |
| `task verify:contract` | Sau thay đổi contract |
| `task verify:full` | Sau mỗi bước migration |
| `python -m logos_eval.flaws` | Đo seeded-flaw recall của critic experiment-design (dùng model thật; không chạy cùng lúc với run live) |
