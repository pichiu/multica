# Multica 開發者上手指南

> 本指南涵蓋從環境建置到 PR 合併的完整開發流程。  
> 閱讀前請確保你已熟悉 `CLAUDE.md` 中的架構與規範。

---

## 目錄

1. [Prerequisites](#1-prerequisites)
2. [環境建置](#2-環境建置)
3. [本地開發 Workflow](#3-本地開發-workflow)
4. [測試策略與執行](#4-測試策略與執行)
5. [Debugging 技巧](#5-debugging-技巧)
6. [Contribution Workflow](#6-contribution-workflow)
7. [Database 操作](#7-database-操作)

---

## 1. Prerequisites

### 必要工具與版本

| 工具 | 最低版本 | 安裝指令（參考） |
|------|----------|-----------------|
| **Node.js** | v20+ | `nvm install 20` |
| **pnpm** | v10.28+ | `npm i -g pnpm@10.28` |
| **Go** | v1.26+ | https://go.dev/dl/ |
| **Docker** | 最新穩定版 | https://docs.docker.com/get-docker/ |
| **sqlc**（可選） | — | `go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest` |

> CI 環境：Node 22、Go 1.26.1、pgvector/pgvector:pg17 PostgreSQL。

---

## 2. 環境建置

### 2.1 Clone 並啟動

```bash
git clone https://github.com/multica-ai/multica.git
cd multica
make dev
```

`make dev` 是一鍵指令，自動完成：
1. 偵測主 checkout 或 worktree，建立正確的 `.env` / `.env.worktree`
2. 檢查 Node.js、pnpm、Go、Docker 是否已安裝
3. 安裝 JavaScript 依賴（`pnpm install`）
4. 啟動共用 PostgreSQL container（`pgvector/pgvector:pg17`）
5. 建立 app database（若不存在）
6. 執行所有 migration
7. 啟動 backend（port 8080）+ frontend（port 3000）

### 2.2 .env 設定說明

主 checkout 複製 example：

```bash
cp .env.example .env
```

#### 最小必要值

```bash
# 資料庫（Docker 預設值，本地開發可直接使用）
DATABASE_URL=postgres://multica:multica@localhost:5432/multica?sslmode=disable

# JWT 金鑰（本地開發可用預設值，生產環境必須更換）
JWT_SECRET=change-me-in-production

# 本地開發：固定 OTP，避免需要真實 email 服務
MULTICA_DEV_VERIFICATION_CODE=888888
```

#### 完整環境變數分類

| 類別 | 變數 | 必要性 | 說明 |
|------|------|--------|------|
| **DB** | `DATABASE_URL` | 必要 | PostgreSQL 連線字串 |
| **Auth** | `JWT_SECRET` | 必要 | JWT 簽名金鑰 |
| **Server** | `PORT` | 可選 | 預設 `8080` |
| **Server** | `ALLOW_SIGNUP` | 可選 | 預設 `true` |
| **Server** | `APP_ENV` | 可選 | `production` 會停用開發驗證碼 |
| **Email** | `RESEND_API_KEY` | 可選 | 不設定則將驗證碼印到 log |
| **OAuth** | `GOOGLE_CLIENT_ID/SECRET` | 可選 | Google 登入 |
| **Storage** | `S3_BUCKET` / `LOCAL_UPLOAD_DIR` | 可選 | 檔案上傳；無 S3 時用本地目錄 |
| **Redis** | `REDIS_URL` | 可選 | 不設定 = 單節點 in-memory 模式 |
| **Dev** | `MULTICA_DEV_VERIFICATION_CODE` | 開發用 | 固定 OTP（`APP_ENV=production` 時忽略）|
| **Frontend** | `NEXT_PUBLIC_API_URL` | 可選 | 留空 = 同域 |

### 2.3 驗證啟動成功

```bash
curl http://localhost:8080/health
# 應回傳 200 OK

curl http://localhost:3000
# 應看到 Next.js 首頁
```

---

## 3. 本地開發 Workflow

### 3.1 日常開發命令

```bash
# 啟動全套服務（自動 migrate）
make start

# 只啟動 Go backend
make server

# 只啟動 Next.js frontend
pnpm dev:web

# 啟動 Electron 桌面應用
pnpm dev:desktop

# 停止當前 checkout 的服務
make stop

# 構建所有前端
pnpm build

# TypeScript 型別檢查
pnpm typecheck

# ESLint
pnpm lint
```

### 3.2 Hot Reload 支援

| 元件 | Hot Reload | 說明 |
|------|-----------|------|
| Next.js web | ✅ | Next.js Fast Refresh |
| Electron desktop | ✅ | electron-vite HMR |
| Go backend | ❌ | 需手動重啟；`make server` |
| Shared packages | ✅ | Internal Packages pattern，bundler 直接編譯 `.ts`/`.tsx`，HMR 零延遲 |

### 3.3 Git Worktree 開發模式

Worktree 讓你同時執行多個功能分支，資料完全隔離：

```bash
# 建立 worktree
git worktree add ../multica-feature -b feat/my-change main

# 進入 worktree 目錄並啟動
cd ../multica-feature
make dev  # 自動生成 .env.worktree，分配唯一 DB 和 port
```

Worktree 與主 checkout 並行執行範例：

| | 主 checkout | Worktree |
|--|------------|----------|
| Database | `multica` | `multica_my_feature_702` |
| Backend port | `8080` | `18782`（hash 衍生）|
| Frontend port | `3000` | `13702`（hash 衍生）|

> **重要**：worktree 目錄中不能有 `.env` 檔，只能有 `.env.worktree`，否則會意外指向主資料庫。

---

## 4. 測試策略與執行

### 測試金字塔

```mermaid
graph TD
    E2E["🎭 E2E（Playwright）<br/>e2e/tests/*.spec.ts<br/>真實瀏覽器 × 真實 backend"]
    INT["🔗 Integration（Go test）<br/>server/internal/handler/*_test.go<br/>handler + 真實 DB"]
    UNIT_TS["⚡ Unit TS（Vitest）<br/>packages/core/ packages/views/<br/>apps/web/"]
    UNIT_GO["⚡ Unit Go（go test）<br/>server/internal/service/<br/>server/internal/util/"]

    E2E --> INT
    INT --> UNIT_TS
    INT --> UNIT_GO

    style E2E fill:#ff6b6b,color:#fff
    style INT fill:#ffa94d,color:#fff
    style UNIT_TS fill:#51cf66,color:#fff
    style UNIT_GO fill:#51cf66,color:#fff
```

### 4.1 測試放置規則

| 測試對象 | 正確位置 | 環境 |
|---------|---------|------|
| 共享業務邏輯（stores、hooks） | `packages/core/*.test.ts` | Node（無 DOM）|
| 共享 UI 元件（頁面、表單） | `packages/views/*.test.tsx` | jsdom + Testing Library |
| Next.js 平台特定行為 | `apps/web/*.test.tsx` | jsdom + 框架 mock |
| 桌面平台行為 | `apps/desktop/src/**/*.test.tsx` | ⚠️ 未驗證 |
| 端到端流程 | `e2e/tests/*.spec.ts` | Playwright |
| Go handler / service | `server/internal/**/*_test.go` | go test + 真實 DB |

### 4.2 執行所有測試

```bash
make check      # 完整驗證：typecheck + TS tests + Go tests + E2E
pnpm typecheck  # 只跑 TypeScript 型別檢查
pnpm test       # 只跑 TS unit tests（Vitest，所有 package）
make test       # 只跑 Go tests
```

### 4.3 執行單一測試

```bash
# 跑特定 views 測試檔
pnpm --filter @multica/views exec vitest run auth/login-page.test.tsx

# 跑特定 core 測試
pnpm --filter @multica/core exec vitest run runtimes/version.test.ts

# 跑特定 web 測試
pnpm --filter @multica/web exec vitest run app/\(auth\)/login/page.test.tsx

# 跑特定 Go 測試
cd server && go test ./internal/handler/ -run TestHandlerName

# 跑特定 E2E 測試（需先啟動 backend + frontend）
pnpm exec playwright test e2e/tests/specific-test.spec.ts
```

---

## 5. Debugging 技巧

### 5.1 Server Log 格式

後端使用 Go 標準 `slog`，輸出結構化 JSON：

```json
{
  "time": "2026-05-05T10:00:00Z",
  "level": "INFO",
  "msg": "request completed",
  "method": "POST",
  "path": "/api/issues",
  "status": 201,
  "duration": "12ms",
  "workspace_id": "..."
}
```

本地開發可用 `jq` 格式化輸出：

```bash
make server 2>&1 | jq -r '"\(.level) \(.msg) \(.path // "")"'
```

### 5.2 開發模式驗證碼

設定 `MULTICA_DEV_VERIFICATION_CODE=888888` 後，所有 OTP 驗證均可使用 `888888`，無需真實 email 服務：

```bash
# .env 中加入
MULTICA_DEV_VERIFICATION_CODE=888888

# 自動化測試流程
curl -X POST http://localhost:8080/auth/send-code \
  -H "Content-Type: application/json" \
  -d '{"email": "dev@localhost"}'

curl -X POST http://localhost:8080/auth/verify-code \
  -H "Content-Type: application/json" \
  -d '{"email": "dev@localhost", "code": "888888"}'
```

> ⚠️ 當 `APP_ENV=production` 時，此設定自動失效。

### 5.3 Codex Sandbox 問題（macOS）

**症狀**：Codex session 中出現 `dial tcp: lookup HOST: no such host`，但在普通 shell 中網路正常。

**原因**：macOS Seatbelt sandbox 的 bug（openai/codex#10390），在 `workspace-write` 模式下阻擋 DNS。

**Daemon 因應措施**：
- macOS + 舊版 Codex：自動降級為 `danger-full-access` 並輸出 WARN 日誌
- Linux（Landlock）：不受影響，正常使用 `workspace-write`

**查看 Daemon 寫入的 sandbox 設定**：

```bash
sed -n '/# BEGIN multica-managed/,/# END multica-managed/p' \
  ~/multica_workspaces/$WORKSPACE_ID/$TASK_SHORT/codex-home/config.toml
```

**解決方案**：升級 Codex CLI（`brew upgrade codex`）。

### 5.4 常見踩坑與解法

| 問題 | 原因 | 解法 |
|------|------|------|
| `Missing env file: .env` | 尚未建立 env 檔 | `cp .env.example .env` |
| `Missing env file: .env.worktree` | Worktree 無 env | `make worktree-env` |
| Worktree 意外使用主 DB | Worktree 目錄有 `.env` | 刪除 `.env`，確認只有 `.env.worktree` |
| Backend 起不來 | Port 已被占用 | `make stop` 或 `lsof -i :8080` |
| Migration 失敗 | DB 不存在 | `make db-up && make migrate-up` |
| Go panic 500 | 未驗證的 UUID 進入 DB 操作 | 使用 `parseUUIDOrBadRequest()` 驗證輸入（見 CLAUDE.md） |

### 5.5 資料庫診斷

```bash
# 列出所有本地 DB
docker compose exec -T postgres psql -U multica -d postgres \
  -At -c "select datname from pg_database order by datname;"

# 確認當前 checkout 使用哪個 DB
grep POSTGRES_DB .env 2>/dev/null || grep POSTGRES_DB .env.worktree

# 完整重置（僅當前 checkout 的 DB）
make stop && make db-reset && make start
```

---

## 6. Contribution Workflow

### 6.1 分支命名規則

```
feat/short-description       # 新功能
fix/bug-description          # Bug 修正
refactor/scope-description   # 重構
docs/what-changed            # 文件
test/scope-description       # 測試
chore/tooling-or-deps        # 工具、依賴
```

### 6.2 Commit 格式（Conventional Commits）

```
feat(scope): 新增某功能
fix(handler): 修正 UUID 驗證邏輯
refactor(core): 提取共享 store 到 packages/core
test(views): 補充 login page 測試
chore(deps): 升級 TanStack Query 至 5.96
```

### 6.3 PR 前置檢查

```bash
make check   # 必跑：typecheck + TS tests + Go tests + E2E
```

部分變更可先跑快速檢查再跑完整：

```bash
pnpm typecheck   # 只改 TS 時
make test        # 只改 Go 時
make check       # 最終驗證
```

### 6.4 CI Pipeline

```mermaid
flowchart LR
    PR[PR 建立/更新] --> CI[GitHub Actions ci.yml]
    CI --> TC[TypeScript typecheck]
    CI --> TT[TS unit tests Vitest]
    CI --> GT[Go tests]
    CI --> E2E[Playwright E2E]
    TC & TT & GT & E2E --> Done[✅ 可合併]

    style PR fill:#4dabf7,color:#fff
    style Done fill:#51cf66,color:#fff
```

CI 環境：
- Node 22、Go 1.26.1
- PostgreSQL：`pgvector/pgvector:pg17` service container
- E2E：backend + frontend 在 CI 中啟動後再跑 Playwright

### 6.5 CLI Release（生產部署前置）

每次 Production 部署都需要搭配 CLI release：

```bash
git tag v0.x.x
git push origin v0.x.x
```

GitHub Actions 自動觸發 `release.yml`：Go tests → GoReleaser 多平台構建 → 發佈到 GitHub Releases + Homebrew tap。

版本號規則：預設 patch bump（`v0.1.12` → `v0.1.13`）。

---

## 7. Database 操作

### 7.1 新增 Migration

Migration 檔案位於 `server/migrations/`，命名格式：`{NNN}_{description}.{up|down}.sql`

```bash
# 1. 建立 migration 檔（手動建立）
touch server/migrations/068_my_new_table.up.sql
touch server/migrations/068_my_new_table.down.sql

# 2. 編寫 SQL
# server/migrations/068_my_new_table.up.sql:
#   CREATE TABLE my_table (...);

# server/migrations/068_my_new_table.down.sql:
#   DROP TABLE IF EXISTS my_table;

# 3. 套用 migration
make migrate-up

# 4. 回滾（如需要）
make migrate-down
```

### 7.2 sqlc 工作流（SQL → 生成 Go）

sqlc 設定於 `server/sqlc.yaml`：
- Query 來源：`server/pkg/db/queries/`
- Schema 來源：`server/migrations/`（所有 `.sql` 檔）
- 輸出目標：`server/pkg/db/generated/`

```bash
# 1. 在 server/pkg/db/queries/ 新增或修改 SQL query
# 例：server/pkg/db/queries/issue.sql

# -- name: GetIssueByID :one
# SELECT * FROM issue WHERE id = $1 AND workspace_id = $2;

# 2. 重新生成 Go 型別
make sqlc

# 3. 使用生成的型別
# server/pkg/db/generated/issue.sql.go 中的 GetIssueByID 函數可直接使用
```

### 7.3 Worktree 多環境 DB 隔離

每個 worktree 有獨立的 DB，共用同一個 PostgreSQL container：

```bash
# 主 checkout
DATABASE_URL=postgres://multica:multica@localhost:5432/multica

# Worktree（自動生成）
DATABASE_URL=postgres://multica:multica@localhost:5432/multica_feat_my_change_702
```

**操作指令**：

```bash
# 生成 worktree env（自動計算唯一 DB 名稱和 port）
make worktree-env

# Worktree 完整初始化
make setup-worktree   # 建立 DB + 執行 migration

# 重置單一 worktree DB（不影響其他 worktree）
make stop-worktree && make db-reset && make start-worktree

# 核查所有 worktree DB
docker compose exec -T postgres psql -U multica -d postgres \
  -At -c "select datname from pg_database where datname like 'multica%' order by datname;"
```

> **警告**：`docker compose down -v` 會刪除共用 Docker volume，所有 worktree 的 DB 資料都會消失。

---

## 快速參考

```bash
# 第一次啟動
make dev

# 日常開發
make start / make stop

# 跑所有測試
make check

# 新增 worktree
git worktree add ../multica-feature -b feat/change main
cd ../multica-feature && make dev

# 套用 migration
make migrate-up

# 重新生成 DB 型別
make sqlc

# 重置 DB（當前 checkout）
make stop && make db-reset && make start
```
