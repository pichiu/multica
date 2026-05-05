# Stage 2.1 Entry Points

## Go Backend Entrypoints

### API Server: `server/cmd/server/main.go`

啟動順序：

1. **環境設定讀取**：`PORT`（預設 8080）、`DATABASE_URL`、`JWT_SECRET`、`REDIS_URL` 等
2. **DB 連線**：`pgxpool` 建立連線池，ping 驗證
3. **事件系統初始化**：`events.New()` → in-process 同步 pub/sub bus
4. **WebSocket Hub**：`realtime.NewHub()` → 用戶 WS 連線管理，`go hub.Run()`
5. **DaemonWS Hub**：`daemonws.NewHub()` → daemon WS 連線管理
6. **Redis Relay（可選）**：若 `REDIS_URL` 存在，初始化 Redis relay，三種模式：
   - `sharded`（預設）：分片 Redis streams，支援 daemon wakeup
   - `dual`：sharded + legacy 雙寫，用於滾動升級
   - `legacy`：舊版單 stream
7. **事件監聽器註冊**：`registerListeners(bus, broadcaster)` — 訂閱 domain 事件 → 廣播到 WS
8. **Analytics**：`analytics.NewFromEnv()` → PostHog client（若 `POSTHOG_API_KEY` 設定）
9. **Chi Router 建立**：`NewRouterWithOptions(...)` → 掛載所有 middleware + routes
10. **Background Workers 啟動**：
    - `runRuntimeSweeper`：標記失線 runtime 為 offline
    - `runAutopilotScheduler`：cron 觸發 autopilot 任務
    - `runDBStatsLogger`：定期記錄 DB 統計
11. **Metrics Server（可選）**：若 `METRICS_ADDR` 設定，啟動 Prometheus HTTP server
12. **HTTP Server 啟動**：`srv.ListenAndServe()`
13. **優雅關閉**：捕捉 `SIGINT`/`SIGTERM`，等待進行中請求完成（10s timeout）

關鍵檔案：
- `server/cmd/server/main.go` — main function
- `server/cmd/server/router.go` — Chi router + 所有路由定義
- `server/internal/handler/handler.go` — Handler struct 定義與建構

### Handler struct (`server/internal/handler/handler.go:46-62`)

```go
type Handler struct {
    Queries               *db.Queries       // sqlc 查詢
    DB                    dbExecutor        // raw pgx connection
    TxStarter             txStarter         // transaction starter
    Hub                   *realtime.Hub     // 用戶 WS hub
    DaemonHub             *daemonws.Hub     // daemon WS hub
    Bus                   *events.Bus       // domain event bus
    TaskService           *service.TaskService
    AutopilotService      *service.AutopilotService
    EmailService          *service.EmailService
    UpdateStore           *UpdateStore      // CLI update request cache
    ModelListStore        ModelListStore    // AI model list cache
    LocalSkillListStore   LocalSkillListStore
    LocalSkillImportStore LocalSkillImportStore
    Storage               storage.Storage  // S3 or local
    CFSigner              *auth.CloudFrontSigner
    Analytics             analytics.Client
    PATCache              *auth.PATCache
    DaemonTokenCache      *auth.DaemonTokenCache
    cfg                   Config
}
```

### CLI & Daemon: `server/cmd/multica/main.go`

Cobra CLI，子命令分組：

**Core 命令（workspace 操作）**：
- `issue` — 建立、列出、更新、查看 issue
- `project` — 專案管理
- `label` — 標籤管理
- `agent` — agent 管理
- `autopilot` — autopilot 管理
- `workspace` — workspace 操作
- `repo` — 倉庫設定
- `skill` — skill 管理

**Runtime 命令**：
- `daemon start/stop/status/restart/logs` — 本地 daemon 生命週期管理
- `runtime` — runtime 列表查詢

**Additional 命令**：
- `auth status/logout` — 認證狀態
- `login` — 瀏覽器 OAuth 或 token 登入
- `setup` — 一鍵配置（含 `self-host` 模式）
- `attachment` — 附件管理
- `config` — 本地設定管理
- `update` — CLI 自我更新
- `version` — 版本資訊

### DB Migrate: `server/cmd/migrate/main.go`

獨立的 migration 執行工具，被 `Makefile` 中的 `make migrate-up` / `make migrate-down` 呼叫。

---

## Next.js Web App Entrypoints

### Root Layout: `apps/web/app/layout.tsx`

- 設定字型：Inter（Latin UI）+ Geist Mono + Source Serif 4
- 掛載 `ThemeProvider`、`Toaster`（Sonner）
- 掛載 `WebProviders`（包含 `CoreProvider`）
- 掛載 `LocaleSync`（locale 同步）

### WebProviders: `apps/web/components/web-providers.tsx`

核心初始化：
- 偵測 legacy localStorage token（cookie auth migration）
- 計算 WS URL（優先 `NEXT_PUBLIC_WS_URL`，否則從 `window.location` 推算）
- 建立 `CoreProvider`：
  - `apiBaseUrl` = `NEXT_PUBLIC_API_URL`
  - `wsUrl` = 計算的 WS URL
  - `cookieAuth` = true（新用戶）或 false（舊 localStorage token 用戶）
  - `onLogin` / `onLogout` = cookie 設定回調
  - `identity` = `{ platform: "web", version: WEB_VERSION }`

### CoreProvider: `packages/core/platform/core-provider.tsx`

Module-level singleton 初始化（只執行一次）：
1. 建立 `ApiClient` 實例並設定 `setApiInstance(api)`
2. 若 token mode，從 storage 讀取 token
3. 建立並註冊 `authStore`（Zustand）
4. 建立並註冊 `chatStore`（Zustand）
5. 包裹 `QueryProvider`（TanStack Query）
6. 掛載 `AuthInitializer`（session 恢復）
7. 掛載 `WSProvider`（WebSocket 連線管理）

### WorkspaceLayout: `apps/web/app/[workspaceSlug]/layout.tsx`

每個 workspace 路由的 guard：
1. 若未認證 → redirect `/login`
2. 用 `workspaceBySlugOptions(slug)` 從 React Query cache 查詢 workspace
3. 呼叫 `setCurrentWorkspace(slug, workspace.id)` — 設定全局 workspace context
4. 寫入 `last_workspace_slug` cookie（用於下次自動跳轉）

---

## Middleware 堆疊 (`server/cmd/server/router.go`)

全局 middleware（所有路由）：
- `chimw.Recoverer` — panic recovery
- `chimw.RequestID` — 請求 ID
- `chimw.RealIP` — 真實 IP
- `corsMiddleware` — CORS（origins 從 env 讀取）
- `obsmetrics.HTTPMetrics.Middleware` — Prometheus 指標（若啟用）

認證保護路由額外 middleware：
- `middleware.AuthOrBadRequest(queries)` — JWT / PAT / daemon token 驗證
  - 支援三種 token 格式：JWT（cookie 或 header）、`mul_` PAT、`mdt_` daemon token
- `middleware.RequireWorkspaceMembership(queries, "X-Workspace-ID")` — workspace 存取控制

---

## Desktop App Entrypoints

位於 `apps/desktop/`，使用 electron-vite。Renderer 進程使用 React Router + 同一套 `@multica/views` 元件。差異：
- Navigation：`react-router-dom` 替代 `next/navigation`
- 多 tab 管理：desktop-only `tab-store`
- pre-workspace 流程：`WindowOverlay` 狀態機替代路由
