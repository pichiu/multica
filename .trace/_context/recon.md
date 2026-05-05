# Stage 1 偵察報告

## 專案定位

**Multica** 是一個開源的 managed agents 平台（Apache 2.0），讓 AI coding agent 成為真正的隊友。
用戶可以像分配任務給同事一樣，將 issue 指派給 agent — agent 自主認領、執行、回報進度、更新狀態。

名稱典故：**Mul**tiplexed **I**nformation and **C**omputing **A**gent，致敬 1960s 時分複用 OS Multics。

- 官網：https://multica.ai
- GitHub：https://github.com/multica-ai/multica（22,000+ stars）
- 支援 Claude Code、Codex、GitHub Copilot CLI、Cursor Agent、Gemini、OpenCode 等

---

## 技術棧

| 類別 | 技術 | 版本 | 備註 |
|------|------|------|------|
| Backend 語言 | Go | 1.26.1 | `server/go.mod` |
| HTTP Framework | Chi | 5.2.5 | `go-chi/chi/v5` |
| WebSocket | gorilla/websocket | 1.5.3 | 實時通知 |
| 資料庫 | PostgreSQL (pgvector/pg17) | 17 | pgx/v5 client |
| ORM/Query | sqlc | — | 從 SQL 自動產生型別 |
| DB Migration | 自訂 migrate tool | — | `server/internal/migrations/` |
| Cache/Pub-Sub | Redis (可選) | — | `go-redis/v9`，多節點部署用 |
| 排程 | robfig/cron | v3 | Autopilot 排程 |
| Email | Resend | — | `resend-go/v2` |
| File Storage | AWS S3 / 本地檔案系統 | — | CloudFront CDN 簽名 |
| Metrics | Prometheus | — | `prometheus/client_golang` |
| Analytics | PostHog | — | 自願開啟 |
| Frontend 語言 | TypeScript | 5.9.x | strict mode |
| Frontend Framework | Next.js (App Router) | — | `apps/web/` |
| Desktop App | Electron (electron-vite) | — | `apps/desktop/` |
| State (Server) | TanStack Query | 5.96.x | React Query |
| State (Client) | Zustand | 5.0.x | client UI state |
| UI Components | shadcn/ui (Base UI) | — | `packages/ui/` |
| CSS | Tailwind CSS | v4 | semantic tokens |
| Testing (TS) | Vitest | 4.1.x | jsdom 環境 |
| Testing (E2E) | Playwright | 1.58.x | `e2e/` |
| Testing (Go) | 標準 `go test` | — | |
| Monorepo | pnpm workspaces + Turborepo | pnpm 10.28 | |
| CLI | cobra | 1.10.x | `server/cmd/multica/` |
| JWT | golang-jwt/jwt | v5 | 身份驗證 |
| ID 生成 | oklog/ulid | v2 | task/session IDs |

---

## 架構模式

**Monorepo（pnpm workspaces + Turborepo）** + **Go backend**

```
multica/
├── apps/
│   ├── web/          # Next.js App Router（主要 web 前端）
│   ├── desktop/      # Electron 桌面應用（electron-vite）
│   └── docs/         # 文件網站（推測）
├── packages/
│   ├── core/         # 無依賴業務邏輯（Zustand stores、React Query hooks、API client）
│   ├── ui/           # Atomic UI 元件（shadcn/Base UI，零業務邏輯）
│   ├── views/        # 共享業務頁面/元件（zero next/*）
│   ├── tsconfig/     # 共享 TypeScript 配置
│   └── eslint-config/ # 共享 ESLint 配置
├── server/           # Go backend
│   ├── cmd/
│   │   ├── server/   # API server main
│   │   └── multica/  # CLI + daemon main
│   ├── internal/
│   │   ├── handler/  # HTTP handlers（Chi router）
│   │   ├── service/  # 業務邏輯（TaskService、AutopilotService）
│   │   ├── daemon/   # 本地 daemon runtime
│   │   ├── daemonws/ # daemon WebSocket hub
│   │   ├── realtime/ # 用戶 WebSocket hub + Redis relay
│   │   ├── events/   # 進程內 pub/sub bus
│   │   ├── auth/     # JWT、PAT、CloudFront 簽名
│   │   ├── middleware/ # Chi middleware（auth、CORS 等）
│   │   ├── analytics/ # PostHog 封裝
│   │   ├── metrics/  # Prometheus
│   │   ├── storage/  # S3 / 本地 storage 介面
│   │   └── util/     # UUID 解析、通用工具
│   ├── pkg/
│   │   ├── db/generated/ # sqlc 自動生成的 DB 型別
│   │   ├── agent/    # agent abstraction layer
│   │   └── protocol/ # daemon ↔ server protocol 定義
│   └── migrations/   # 68 個 migration 檔案（001-067）
├── e2e/              # Playwright E2E 測試
├── docker/           # Docker 相關設定
└── scripts/          # 安裝腳本（install.sh、install.ps1）
```

---

## 核心概念

### Agent 核心抽象

- **Agent**：workspace 中的 AI 執行者，有 profile（名稱、avatar、狀態）
- **Runtime**：compute 環境，向 server 註冊可用的 AI CLI
- **Daemon**：本地 runtime，掃描 PATH 偵測已安裝的 CLI，輪詢任務
- **Task**：`agent_task_queue` 中的工作單元，狀態機：`queued → dispatched → running → completed/failed/cancelled`
- **Skill**：知識包（SKILL.md + 支援檔案），注入每次任務執行的 context
- **Autopilot**：cron/觸發式自動化，無需人工分配即可派發 agent 任務

### 資料模型（主要 DB 表）

已確認的 table（`server/migrations/*.up.sql`）：

- `user`、`workspace`、`member` — 多租戶基礎
- `agent`、`agent_runtime`、`agent_skill` — agent 系統
- `issue`、`issue_label`、`issue_to_label`、`issue_dependency`、`issue_subscriber` — 任務追蹤
- `comment`、`comment_reaction`、`issue_reaction` — 協作
- `agent_task_queue`、`task_message`、`task_usage` — 任務執行
- `skill`、`skill_file` — 技能系統
- `autopilot`、`autopilot_run`、`autopilot_trigger` — 自動化
- `chat_session`、`chat_message` — 直接對話
- `project`、`project_resource` — 專案管理
- `inbox_item`、`notification_preference` — 通知
- `personal_access_token`、`daemon_token`、`verification_code` — 認證
- `pinned_item`、`activity_log`、`attachment` — 輔助功能
- `workspace_invitation`、`feedback` — 成長/社群

---

## 部署模式

1. **Cloud（multica.ai）**：官方托管，agent 在用戶本機 daemon 執行
2. **Self-hosted（Docker Compose）**：單機 `docker-compose.selfhost.yml`，從 GHCR 拉取鏡像
3. **Self-hosted Advanced**：Kubernetes / 多節點（Redis relay + 多 API pods）

---

## 既有文件清單與品質評估

| 文件 | 路徑 | 品質 | 備註 |
|------|------|------|------|
| README（EN） | `README.md` | ★★★★★ | 完整，有安裝、功能、Quick Start |
| README（ZH-CN） | `README.zh-CN.md` | ★★★★☆ | 簡體中文版本 |
| 產品全景文件 | `docs/product-overview.md` | ★★★★★ | 詳盡的中文功能說明，截至 2026-04-21 |
| CLI & Daemon | `CLI_AND_DAEMON.md` | ★★★★★ | 完整的 CLI 指令參考 |
| CLI 安裝 | `CLI_INSTALL.md` | ★★★★☆ | 多平台安裝步驟 |
| 貢獻指南 | `CONTRIBUTING.md` | ★★★★★ | 詳盡的本地開發流程 |
| Self-hosting | `SELF_HOSTING.md` | ★★★★★ | Docker Compose 部署 |
| Self-hosting Advanced | `SELF_HOSTING_ADVANCED.md` | ★★★★☆ | Kubernetes、多節點 |
| Self-hosting AI | `SELF_HOSTING_AI.md` | ★★★☆☆ | 簡短 |
| Agents Guide | `AGENTS.md` | ★★★★☆ | AI agent 使用指南（指向 CLAUDE.md） |
| Dev Rules | `CLAUDE.md` | ★★★★★ | 最完整的架構與開發規則文件 |
| Design | `docs/design.md` | ★★★★☆ | 產品設計決策 |
| Docs Outline | `docs/docs-outline.md` | 草稿 | 文件重寫計畫 |
| Analytics | `docs/analytics.md` | ★★★☆☆ | PostHog 分析記錄 |

---

## 既有文件 vs 程式碼落差分析

以下僅記錄發現的落差，非批評：

1. **`docs/docs-outline.md` 和 `docs/docs-rewrite-plan.md`**：這些是文件重寫計畫草稿，表示官方文件仍在重組中。文件說有些功能還沒文件，這與程式碼中已存在的功能（如 Autopilot、Chat、Projects）一致。

2. **`docs/product-overview.md` 是簡體中文**：儘管 README 是英文優先，這份詳盡的產品文件是簡體中文撰寫（非台灣繁體），且截止於 2026-04-21，可能未反映最新功能。

3. **`SELF_HOSTING_AI.md`** 極簡，但程式碼中已支援多種 AI backend（Claude Code、Codex、Gemini、Cursor 等），文件覆蓋不足。

4. **Migration 68 個但文件無 schema 說明**：`migrations/` 有 68 個 migration，但沒有對應的 schema 文件。`docs/` 下沒有 `schema.md` 或類似文件。

---

## CI/CD

- `.github/workflows/ci.yml`：Node 22 + Go 1.26.1 + pgvector/pgvector:pg17 PostgreSQL 服務
- `.goreleaser.yml`：多平台二進制構建，發佈到 GitHub Releases + Homebrew tap
- Vercel：前端部署（`.vercelignore` 存在）
- Docker 鏡像：`ghcr.io/multica-ai/multica-backend` 和 `ghcr.io/multica-ai/multica-web`

---

## 環境變數關鍵設定

```
DATABASE_URL          # PostgreSQL 連線（必要）
JWT_SECRET            # 身份驗證（必要，生產環境）
REDIS_URL             # 可選；有此變數才啟用多節點 Redis relay
RESEND_API_KEY        # Email 驗證碼（無則印 log）
GOOGLE_CLIENT_ID/SECRET # OAuth（可選）
S3_BUCKET/REGION      # 檔案上傳（無則用本地 LOCAL_UPLOAD_DIR）
CLOUDFRONT_*          # 簽名 URL（可選）
POSTHOG_API_KEY       # Analytics（可選）
ALLOW_SIGNUP          # 控制是否開放公開註冊
MULTICA_DEV_VERIFICATION_CODE # 本地開發跳過 email 驗證碼
APP_ENV               # 環境識別（production 會影響部分行為）
```
