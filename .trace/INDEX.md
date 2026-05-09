# Multica 專案總覽

## 一句話摘要

Multica 是一個開源的 **managed agents 平台**，讓 AI coding agent（Claude Code、Codex 等）成為真正的團隊成員——分配 issue、自主執行、回報進度、累積技能。設計給 2-10 人的 AI-native 小團隊使用。

---

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Backend 語言 | Go | 1.26.1 | API server + CLI + daemon |
| HTTP Framework | Chi | 5.2.5 | REST API routing + middleware |
| WebSocket | gorilla/websocket | 1.5.3 | 實時推送（用戶 + daemon） |
| 資料庫 | PostgreSQL | 17 (pgvector) | 主資料庫（sqlc 型別安全查詢） |
| Cache / Pub-Sub | Redis | — | 多節點 relay（可選） |
| 排程 | robfig/cron | v3 | Autopilot 定時任務 |
| 排程（DB） | pg_cron | — | task_usage_daily 每小時彙整任務 |
| Email | Resend | — | OTP 驗證碼 |
| 檔案儲存 | AWS S3 / 本地 | — | 附件上傳 |
| Frontend | Next.js App Router | — | 主 Web 應用 |
| Desktop App | Electron + electron-vite | — | 跨平台桌面應用 |
| Server State | TanStack Query | 5.x | API 資料快取 + 樂觀更新 |
| Client State | Zustand | 5.x | UI 狀態管理 |
| UI Components | shadcn/ui (Base UI) | — | Atomic UI 元件庫 |
| CSS | Tailwind CSS | v4 | 樣式系統（semantic tokens） |
| i18n | i18next (via packages/core/i18n/) | — | 多語言支援（en + zh-Hans） |
| Monorepo | pnpm workspaces + Turborepo | — | 前端 monorepo 管理 |
| Testing (TS) | Vitest | 4.x | 單元 / 整合測試 |
| Testing (E2E) | Playwright | 1.58.x | 端到端測試 |
| Testing (Go) | 標準 go test | — | Go 單元測試 |
| Analytics | PostHog | — | 用戶行為追蹤（可選） |
| Metrics | Prometheus | — | 性能監控（可選） |

---

## 關鍵指令速查

```bash
# 一鍵啟動（推薦）
make dev              # 自動建立環境 + 啟動 DB + 遷移 + 啟動應用

# 個別啟動
make setup            # 首次：初始化 DB + 執行 migration
make start            # 啟動 backend + frontend
make stop             # 停止應用程序
make db-down          # 停止 PostgreSQL 容器

# 前端
pnpm dev:web          # Next.js 開發服務器（port 3000）
pnpm dev:desktop      # Electron 開發（HMR）
pnpm build            # 構建所有前端應用
pnpm typecheck        # TypeScript 型別檢查
pnpm lint             # ESLint
pnpm test             # TS 單元測試（Vitest）

# 後端
make server           # 執行 Go server（port 8080）
make daemon           # 執行本地 daemon
make build            # 構建二進制到 server/bin/
make test             # Go 測試
make sqlc             # 重新生成 sqlc 程式碼
make migrate-up       # 執行資料庫 migration
make migrate-down     # 回滾 migration

# 完整驗證（推送前必跑）
make check            # typecheck + TS tests + Go tests + E2E

# CLI
multica setup         # 一鍵配置（連線 Cloud + 登入 + 啟動 daemon）
multica daemon start  # 啟動本地 daemon
multica issue list    # 列出 issues
multica login         # 瀏覽器 OAuth 登入
multica workspace update  # 更新 workspace 設定
multica daemon disk-usage # 查看 daemon 磁碟使用量
```

---

## 文件地圖

| 文件 | 說明 |
|------|------|
| [INDEX.md](INDEX.md) | 本文件：專案總覽與速查 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | 系統架構、元件關係、設計決策 |
| [DATA_MODEL.md](DATA_MODEL.md) | 資料模型、ER 圖、狀態機 |
| [API_SURFACE.md](API_SURFACE.md) | REST API 概覽、認證授權、關鍵 endpoint 詳情 |
| [API_SURFACE_part2.md](API_SURFACE_part2.md) | WebSocket 事件（47 個）、Daemon API、Error Handling |
| [DEV_GUIDE.md](DEV_GUIDE.md) | 開發者上手指南 |
| [CODEBASE_MAP.md](CODEBASE_MAP.md) | 程式碼地圖、目錄說明 |
| [DISCOVERY_LOG.md](DISCOVERY_LOG.md) | 探索紀錄、技術債、待解問題 |

---

## 專案專屬術語表

| 術語 | 定義 |
|------|------|
| **Agent** | Multica 中的 AI 執行者，有 profile（名稱、avatar、狀態），可被指派 issue |
| **Runtime** | 向 server 註冊的 compute 環境（e.g. 本地 daemon），列出可用的 AI CLI |
| **Daemon** | 本地 runtime，是 Multica CLI 的一部分，偵測 AI CLI、輪詢並執行任務 |
| **Task** | `agent_task_queue` 中的工作單元，有明確的狀態機（queued→dispatched→running→completed/failed） |
| **Skill** | 知識包（SKILL.md + 支援檔案），注入每次任務的 context，讓能力可複用 |
| **Autopilot** | cron 或 webhook 觸發的自動化，無需人工分配即可派發 agent 任務 |
| **Issue** | 任務管理單元（類似 GitHub Issues），是 agent 執行的核心載體 |
| **Workspace** | 多租戶隔離單元，每個 workspace 有自己的 agents、issues、settings |
| **Member** | workspace 中的人類用戶，角色：owner / admin / member |
| **Session** | AI CLI 的會話 ID，用於跨任務恢復上下文（不從頭開始） |
| **Inbox** | 用戶的通知中心，接收 issue 更新、任務完成、agent 提問等通知 |
| **Chat** | 直接與 agent 對話的介面（非 issue 驅動的任務） |
| **Provider** | AI CLI 類型識別符（`"claude"`、`"codex"`、`"cursor"` 等） |
| **PAT** | Personal Access Token，格式 `mul_xxx`，用於 API 認證 |
| **Daemon Token** | Daemon 專用認證 token，格式 `mdt_xxx`，綁定特定 workspace |
| **Slug** | workspace 的 URL-friendly 識別符（e.g. `my-team`），用於 URL path |
| **Issue Prefix** | workspace 的 issue 前綴（e.g. `MUL`），issue 識別符 = `MUL-123` |
| **execenv** | 每個任務的隔離工作目錄，包含 TASK.md、skill 檔案、context |
| **Worktree** | git worktree，每個功能分支的獨立 checkout，有自己的 DB 和端口 |
| **i18n Namespace** | 翻譯資源的分類單元（21 個：agents、auth、issues 等），每個 namespace 對應一個 JSON 檔，存放於 `packages/views/locales/<lang>/` |
| **task_usage_daily** | `task_usage` 的日彙整物化表，以 pg_cron 每小時更新，用於 ListRuntimeUsage 查詢效能優化 |
