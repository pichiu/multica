# Stage 2.4 Extension Points

## 1. Agent Backend 擴充（最核心的擴充點）

**位置**：`server/pkg/agent/agent.go`

```go
type Backend interface {
    Execute(ctx context.Context, prompt string, opts ExecOptions) (*Session, error)
}
```

**如何新增一個 AI CLI 支援**：
1. 在 `server/pkg/agent/` 建立新檔案（e.g. `myagent.go`）
2. 實作 `Backend` interface：建立 CLI subprocess，stream stdout 到 `Messages` channel，完成後送 `Result`
3. 在 `agent.go:New()` 的 switch 中新增 `case "myagent": return NewMyAgent(cfg), nil`
4. 在 `server/internal/daemon/identity.go` 新增 CLI 偵測邏輯（掃描 PATH）

**現有 11 個 Backend**：claude, codex, copilot, opencode, openclaw, hermes, gemini, pi, cursor, kimi, kiro

---

## 2. Storage 後端擴充

**位置**：`server/internal/storage/storage.go`

```go
type Storage interface {
    Upload(ctx context.Context, key string, data []byte, contentType, filename string) (string, error)
    Delete(ctx context.Context, key string)
    DeleteKeys(ctx context.Context, keys []string)
    KeyFromURL(rawURL string) string
    CdnDomain() string
}
```

**現有實作**：
- `S3Storage`：AWS S3 + CloudFront CDN（生產環境）
- `LocalStorage`：本地檔案系統（開發 / self-hosted 無 S3 時）

**如何新增**：實作 `Storage` interface，在 `router.go` 初始化邏輯中加入判斷。

---

## 3. Analytics Client 擴充

**位置**：`server/internal/analytics/client.go`

```go
type Client interface {
    // 各種追蹤事件方法
    TrackSignup(userID string, props map[string]any)
    // ...
    Close()
}
```

**現有實作**：
- `PostHogClient`：PostHog 事件追蹤（生產環境）
- `NoopClient`：空操作（開發 / 未設定 API key 時）

---

## 4. Event Bus 監聽器（進程內 Hook）

**位置**：`server/internal/events/bus.go`

```go
// 訂閱特定事件類型
bus.Subscribe("issue:updated", func(e events.Event) {
    // 處理邏輯
})

// 訂閱所有事件
bus.SubscribeAll(func(e events.Event) {
    // 全量監聽
})
```

**現有監聽器群組**（`server/cmd/server/router.go`）：
- `registerListeners` — WS 廣播
- `registerSubscriberListeners` — 訂閱者記錄（必須先於 notification）
- `registerActivityListeners` — 活動日誌
- `registerNotificationListeners` — inbox 通知
- `registerAutopilotListeners` — autopilot 完成觸發

**如何新增**：在 router.go 的 `registerXxxListeners` 函式中新增 `bus.Subscribe(EventType, handler)` 呼叫。

---

## 5. Realtime Broadcaster 擴充

**位置**：`server/internal/realtime/` + `daemonws/`

```go
type Broadcaster interface {
    Broadcast(workspaceID string, payload []byte)
}
```

**現有實作**：
- `Hub`：in-memory（單節點）
- `DualWriteBroadcaster`：Hub + Redis relay（多節點）
- `ShardedStreamRelay`：分片 Redis streams（高吞吐）
- `MirroredRelay`：雙寫（滾動升級期間）

**如何新增**：實作 `Broadcaster` 或 `ManagedRelay` interface。

---

## 6. Realtime 範圍授權（ScopeAuthorizer）

**位置**：`server/internal/realtime/hub.go`

```go
type ScopeAuthorizer interface {
    AuthorizeScope(ctx context.Context, userID, workspaceID, scopeType, scopeID string) (bool, error)
}
```

用於限制 WS 連線可以訂閱哪些細粒度 scope（e.g. 特定 task、特定 chat session）。

---

## 7. PAT Cache / Daemon Token Cache

**位置**：`server/internal/auth/`

可配置的快取層（Redis-backed 或 nil = 每次 DB 查詢）：

```go
type PATCache struct { ... }      // Personal Access Token 快取
type DaemonTokenCache struct { .. } // Daemon Token 快取
```

---

## 8. Frontend Navigation Adapter（跨平台路由抽象）

**位置**：`packages/views/platform/`、`apps/web/platform/navigation.tsx`、`apps/desktop/src/renderer/src/platform/navigation.tsx`

```ts
// packages/views 中使用
const nav = useNavigation()
nav.push('/workspaces/new')  // 不直接調用 next/router 或 react-router
```

**兩個實作**：
- Web：`next/navigation` 的 `useRouter()`
- Desktop：`react-router-dom` 的 `useNavigate()`，並攔截 pre-workspace 路徑轉為 `WindowOverlay`

**如何新增平台**：實作 `NavigationAdapter` interface，提供自己的 `NavigationProvider`。

---

## 9. StorageAdapter（前端持久化抽象）

**位置**：`packages/core/types/storage.ts`

```ts
interface StorageAdapter {
    getItem(key: string): string | null
    setItem(key: string, value: string): void
    removeItem(key: string): void
}
```

讓 `packages/core` 不直接依賴 `localStorage`，不同平台可注入不同實作（Web localStorage、Electron IPC 持久化等）。

---

## 10. CoreProvider 回調 Hooks

**位置**：`packages/core/platform/core-provider.tsx`

```ts
<CoreProvider
    onLogin={() => { /* 平台特定的登入後動作 */ }}
    onLogout={() => { /* 平台特定的登出後動作 */ }}
    identity={{ platform: "web", version: "1.0" }}
>
```

各平台可注入不同的 `onLogin`/`onLogout` 行為（Web 設定 cookie，Desktop 可能保存 token 到系統 keychain）。

---

## 11. Desktop WindowOverlay（pre-workspace 流程擴充點）

**位置**：`apps/desktop/src/renderer/src/stores/window-overlay-store.ts`

```ts
type WindowOverlayType = 'new-workspace' | 'accept-invite' | ...
```

新增 pre-workspace 流程（不走路由，而是全屏 overlay）：
1. 在 `window-overlay-store.ts` 新增 overlay type
2. 在 `navigation.tsx` 中攔截對應路徑，dispatch `setWindowOverlay(type)`
3. 在 overlay 容器中渲染對應的 view component

---

## 整體擴充設計原則

Multica 的擴充點設計遵循兩個核心模式：

**Go 端（介面 + 工廠）**：
- 介面定義在獨立的 `interfaces.go` 或主體檔案中
- 工廠函式（`New*`）根據環境變數或參數選擇實作
- 測試時注入 mock 實作

**TypeScript 端（Adapter + Provider）**：
- 跨平台行為抽象為 Adapter（NavigationAdapter、StorageAdapter）
- 平台特定實作在各 `app/platform/` 目錄
- `CoreProvider` 統一初始化，透過 props 注入平台行為
