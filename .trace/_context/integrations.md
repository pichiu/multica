# Stage 2.5 外部整合

## 1. AI Coding CLIs（核心整合）

這是 Multica 最重要的整合，每個 CLI 都是一個 `agent.Backend` 實作。

| Provider | 執行方式 | 設定 |
|----------|---------|------|
| Claude Code | `claude --print "<prompt>"` | `MULTICA_CLAUDE_PATH`, `--model` |
| Codex | `codex run "<prompt>"` | `MULTICA_CODEX_PATH`, `--model`, sandbox 支援 |
| GitHub Copilot CLI | `gh copilot ...` | CLI binary path |
| Cursor Agent | 平台特定調用 | 系統程序 |
| Gemini CLI | `gemini ...` | CLI binary |
| OpenCode | `opencode ...` | CLI binary |
| OpenClaw | `openclaw ...` | CLI binary |
| Hermes | `hermes ...` | CLI binary |
| Kimi | `kimi ...` | CLI binary |
| Kiro CLI | `kiro-cli ...` | CLI binary |
| Pi | `pi ...` | CLI binary |

**失敗處理**：
- AI CLI 返回非零退出碼 → `result.Status = "failed"`
- 語義無活動 timeout（`SemanticInactivityTimeout`）→ `result.Status = "timeout"`
- Daemon 每 5s 輪詢取消狀態 → 若被取消，SIGTERM 發送給 CLI subprocess

---

## 2. PostgreSQL（pgvector/pg17）

**客戶端**：`github.com/jackc/pgx/v5`（pgxpool 連線池）
**查詢生成**：sqlc（`server/pkg/db/generated/`），從 `server/pkg/db/queries/*.sql` 生成

**連線設定**：
```
DATABASE_URL=postgres://user:pass@host:5432/dbname?sslmode=disable
```

**連線池**：pgxpool，設定在 `server/cmd/server/db.go`。

---

## 3. Redis（可選）

**客戶端**：`github.com/redis/go-redis/v9`
**用途**：

| 用途 | Redis Client 名稱 | 說明 |
|------|-----------------|------|
| Realtime relay 寫入 | `realtime-write` | 發佈 WS 事件到 Redis stream |
| Realtime relay 讀取 | `realtime-read` / `realtime-read-sharded` | 消費 Redis stream |
| 本地 skill 快取 | `store` | 跨節點共享 pending 請求 |
| PAT cache | `store` | Personal Access Token 快取 |
| Daemon token cache | `store` | daemon 認證 token 快取 |
| EmptyClaimCache | `store` | "此 runtime 無任務"快取，優化 claim 路徑 |

**設定**：`REDIS_URL=redis://localhost:6379`
**不設定時**：所有快取使用 in-memory，realtime 為 in-process Hub（單節點）

---

## 4. AWS S3 + CloudFront（檔案上傳）

**客戶端**：`github.com/aws/aws-sdk-go-v2/service/s3`
**用途**：用戶上傳的附件（issue attachments）

**設定**：
```
S3_BUCKET=my-bucket
S3_REGION=us-west-2
CLOUDFRONT_KEY_PAIR_ID=...
CLOUDFRONT_PRIVATE_KEY_SECRET=multica/cloudfront-signing-key  # AWS Secrets Manager
CLOUDFRONT_PRIVATE_KEY=...  # 直接設定（替代 Secrets Manager）
CLOUDFRONT_DOMAIN=cdn.example.com
```

**Fallback**：若 S3 未設定，回退到 `LocalStorage`（本地檔案系統）。

---

## 5. AWS Secrets Manager（CloudFront 金鑰）

**客戶端**：`github.com/aws/aws-sdk-go-v2/service/secretsmanager`
**用途**：從 Secrets Manager 讀取 CloudFront private key（可選）

---

## 6. Resend（Email）

**客戶端**：`github.com/resend/resend-go/v2`
**用途**：發送 email 驗證碼（OTP 登入）

**設定**：
```
RESEND_API_KEY=re_xxx
RESEND_FROM_EMAIL=noreply@multica.ai
```

**Fallback**：若未設定 `RESEND_API_KEY`，驗證碼直接印到 server log（開發模式）。

---

## 7. Google OAuth

**用途**：第三方登入（Google 帳號）
**流程**：
1. 前端呼叫 `POST /auth/google { code, redirect_uri }`
2. Server 向 `https://oauth2.googleapis.com/token` 交換 access token
3. 用 access token 向 `https://www.googleapis.com/oauth2/v2/userinfo` 取得 user info
4. 建立或更新 `user` 記錄，簽發 JWT

**設定**：
```
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=http://localhost:3000/auth/callback
```

**Fallback**：若未設定，`/auth/google` 返回 503（未配置）。

---

## 8. PostHog（Analytics）

**客戶端**：Go server 端 + 前端 `posthog-js`
**用途**：用戶行為追蹤（可選，可完全關閉）

**設定**：
```
POSTHOG_API_KEY=phc_xxx
POSTHOG_HOST=https://us.i.posthog.com
ANALYTICS_DISABLED=true  # 完全關閉
```

**Fallback**：若未設定 API key，使用 `analytics.NoopClient{}` — 零操作。

---

## 9. Prometheus（Metrics）

**客戶端**：`github.com/prometheus/client_golang`
**指標包含**：HTTP request rate/latency、realtime WS 連線數、daemon WS 連線數、DB pool 統計、版本資訊

**設定**：
```
METRICS_ADDR=:9090           # 若設定則啟動 metrics server
REALTIME_METRICS_TOKEN=xxx   # /health/realtime 端點的 bearer token
```

---

## 10. GitHub（Repo 整合）

**整合方式**：非直接 API 整合 — 而是透過 Project Resources 機制
- 用戶在 Project 設定中新增 `github_repo` 類型的 resource（填入 repo URL）
- daemon 在任務執行時取得這些 URL，執行 `git clone` / `git worktree add`
- AI CLI 在本地 git 工作目錄中工作

**repo cache** (`server/internal/daemon/repocache/`)：
- bare clone 快取（`{workspacesRoot}/.repos/`）
- 任務執行時建立 worktree，避免重複 clone

---

## 11. Vercel（前端部署）

**設定**：`.vercelignore`（排除 Go server、desktop app 等）
**環境變數**：
```
NEXT_PUBLIC_API_URL=https://api.multica.ai
NEXT_PUBLIC_WS_URL=wss://api.multica.ai/ws
NEXT_PUBLIC_APP_VERSION=v0.2.0
```

---

## 整合失敗處理總結

| 整合 | 失敗影響 | 處理方式 |
|------|---------|---------|
| PostgreSQL | 致命 — server 無法啟動 | `os.Exit(1)` |
| Redis | 降級 — 單節點模式 | log warning，繼續啟動 |
| S3 | 降級 — 本地 storage | 自動 fallback |
| Resend | 降級 — 驗證碼印 log | log warning |
| Google OAuth | 功能不可用 | API 返回 503 |
| PostHog | 靜默降級 | NoopClient |
| Prometheus | 功能不啟動 | 僅 log warning |
| AI CLI | 任務失敗 | `FailTask()` 記錄錯誤 |
