# Stage 2.6 設定與環境

## 設定優先順序

```
環境變數 > .env / .env.worktree 檔案 > 程式碼預設值
```

Server 直接讀取 `os.Getenv()`，無配置檔格式（無 YAML/TOML config 檔）。

CLI daemon 有自己的 TOML config 檔（詳見下方）。

---

## 環境變數完整清單

### 必要設定

| 變數 | 說明 | 預設值 |
|------|------|--------|
| `DATABASE_URL` | PostgreSQL 連線字串 | `postgres://multica:multica@localhost:5432/multica?sslmode=disable` |
| `JWT_SECRET` | JWT 簽名金鑰 | `change-me-in-production`（不安全，會 log warning） |

### Server 核心設定

| 變數 | 說明 | 預設值 |
|------|------|--------|
| `PORT` | HTTP 監聽端口 | `8080` |
| `APP_ENV` | 環境標識（`production` 會改變部分行為） | 空 |
| `ALLOW_SIGNUP` | 是否允許公開註冊 | `true` |
| `ALLOWED_EMAILS` | 白名單 email（逗號分隔） | 空（無限制） |
| `ALLOWED_EMAIL_DOMAINS` | 白名單 email domain | 空 |
| `ALLOWED_ORIGINS` / `CORS_ALLOWED_ORIGINS` | CORS 允許來源 | localhost:3000,5173,5174 |
| `FRONTEND_ORIGIN` | 單一前端 origin | 空 |

### 認證設定

| 變數 | 說明 |
|------|------|
| `GOOGLE_CLIENT_ID` | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret |
| `GOOGLE_REDIRECT_URI` | OAuth 回調 URI |

### Redis 設定

| 變數 | 說明 | 效果 |
|------|------|------|
| `REDIS_URL` | Redis 連線 URL | 不設定 = 單節點 in-memory 模式 |
| `REALTIME_RELAY_MODE` | `sharded`/`dual`/`legacy` | 預設 `sharded` |
| `REALTIME_RELAY_SHARDS` | stream 分片數 | 預設 4 |
| `REALTIME_RELAY_STREAM_MAXLEN` | 每個 stream 最大長度 | 預設 10000 |

### 檔案上傳設定

| 變數 | 說明 |
|------|------|
| `S3_BUCKET` | S3 bucket 名稱 |
| `S3_REGION` | AWS region（預設 `us-west-2`） |
| `LOCAL_UPLOAD_DIR` | 本地上傳目錄（預設 `./data/uploads`） |
| `LOCAL_UPLOAD_BASE_URL` | 本地上傳的 base URL |
| `CLOUDFRONT_KEY_PAIR_ID` | CloudFront 簽名金鑰 ID |
| `CLOUDFRONT_PRIVATE_KEY` | CloudFront private key（直接） |
| `CLOUDFRONT_PRIVATE_KEY_SECRET` | CloudFront key 的 Secrets Manager 名稱 |
| `CLOUDFRONT_DOMAIN` | CDN domain |

### Email 設定

| 變數 | 說明 |
|------|------|
| `RESEND_API_KEY` | Resend API key（不設定 = 印 log） |
| `RESEND_FROM_EMAIL` | 發件人 email（預設 `noreply@multica.ai`） |

### Analytics / Metrics

| 變數 | 說明 |
|------|------|
| `POSTHOG_API_KEY` | PostHog API key |
| `POSTHOG_HOST` | PostHog host（預設 `https://us.i.posthog.com`） |
| `ANALYTICS_DISABLED` | 設定任意值即禁用 analytics |
| `METRICS_ADDR` | Prometheus metrics 監聽地址（e.g. `:9090`） |
| `REALTIME_METRICS_TOKEN` | `/health/realtime` 端點的 bearer token |

### 開發專用設定

| 變數 | 說明 | 注意 |
|------|------|------|
| `MULTICA_DEV_VERIFICATION_CODE` | 固定的 OTP 驗證碼 | 生產環境 + APP_ENV=production 時自動忽略 |
| `COOKIE_DOMAIN` | Session cookie domain | 跨域部署用 |

---

## CLI Daemon 設定

Daemon 使用獨立的 TOML config 檔（`server/internal/cli/config.go`）：

**設定檔位置**：
- macOS：`~/.config/multica/config.toml`
- Linux：`~/.config/multica/config.toml`
- Windows：`%APPDATA%\multica\config.toml`

**設定內容**：
```toml
# server 連線
server_url = "https://api.multica.ai"

# 認證 token
token = "mul_..."

# 已配置的 workspace
[[workspaces]]
id = "uuid"
slug = "my-workspace"

# daemon 行為
[daemon]
poll_interval = "3s"
heartbeat_interval = "15s"
workspaces_root = "~/multica_workspaces"
```

**CLI 環境變數覆蓋**：

| 變數 | 說明 |
|------|------|
| `MULTICA_SERVER_URL` | server URL 覆蓋 |
| `MULTICA_WORKSPACE_ID` | workspace ID |
| `MULTICA_DAEMON_POLL_INTERVAL` | 輪詢間隔（預設 `3s`） |
| `MULTICA_DAEMON_HEARTBEAT_INTERVAL` | heartbeat 間隔（預設 `15s`） |
| `MULTICA_CODEX_PATH` | Codex CLI 路徑 |
| `MULTICA_CODEX_MODEL` | Codex model |
| `MULTICA_CODEX_WORKDIR` | Codex 預設工作目錄 |
| `MULTICA_CODEX_TIMEOUT` | Codex 超時（預設 `20m`） |

---

## 前端環境變數

**Build-time（編譯時注入）**：

| 變數 | 說明 |
|------|------|
| `NEXT_PUBLIC_API_URL` | API server URL（留空 = 同域） |
| `NEXT_PUBLIC_WS_URL` | WebSocket URL（留空 = 自動從 window.location 推算） |
| `NEXT_PUBLIC_APP_VERSION` | 版本字串（CI 設定） |

---

## 多環境隔離（Worktree 模式）

主 checkout 使用 `.env`，git worktree 使用 `.env.worktree`：

```bash
make worktree-env  # 自動生成 .env.worktree，分配唯一 DB 名稱和端口
```

**工作原理**：
- `POSTGRES_DB=multica_{branch_hash}` — 唯一 DB 名稱
- `PORT=1xxxx` — hash 衍生的唯一 backend 端口
- `FRONTEND_PORT=1xxxx` — hash 衍生的唯一 frontend 端口
- 共用同一個 PostgreSQL container（port 5432）

---

## Feature Flags

Multica 無專門的 feature flag 系統。功能開關透過：

1. **環境變數**：`ALLOW_SIGNUP`、`ANALYTICS_DISABLED`
2. **DB 設定**（`workspace.settings` JSONB）：workspace 級別的設定
3. **程式碼條件**：`APP_ENV=production` 控制部分行為（如禁用開發驗證碼）

---

## Secrets 管理

**本地開發**：明文 `.env` 檔（git ignored）
**生產環境**：
- AWS Secrets Manager（CloudFront private key）
- 環境變數注入（容器 / K8s secrets）

**安全注意**：
- `JWT_SECRET` 必須隨機且足夠長
- `mul_` PAT token 有快取，`DaemonTokenCache` 和 `PATCache` 用 Redis 快取 token 驗證結果

---

## Docker Compose 配置

**開發**：`docker-compose.yml`（僅 PostgreSQL）
**Self-hosted 生產**：`docker-compose.selfhost.yml`（完整 stack：PostgreSQL + backend + web）
**Self-hosted 構建版**：`docker-compose.selfhost.build.yml`（本地構建鏡像）

**鏡像**：
```
MULTICA_BACKEND_IMAGE=ghcr.io/multica-ai/multica-backend
MULTICA_WEB_IMAGE=ghcr.io/multica-ai/multica-web
MULTICA_IMAGE_TAG=latest
```
