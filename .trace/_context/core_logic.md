# Stage 2.3 核心領域邏輯

## 核心抽象：Agent Backend Interface

Multica 最核心的非 trivial 設計是 **統一 agent 抽象層**（`server/pkg/agent/agent.go`）。

```go
// Backend 是所有 AI CLI 的統一介面
type Backend interface {
    Execute(ctx context.Context, prompt string, opts ExecOptions) (*Session, error)
}
```

**已實作的 Backend**（`server/pkg/agent/`）：
- `claude.go` — Anthropic Claude Code (`claude --print`)
- `codex.go` — OpenAI Codex CLI（含 sandbox 支援）
- `copilot.go` — GitHub Copilot CLI
- `cursor.go` — Cursor Agent（多平台調用策略）
- `gemini.go` — Google Gemini CLI
- `hermes.go` — Hermes（社群 agent）
- `kimi.go` — Kimi CLI
- `kiro.go` — Kiro CLI
- `openclaw.go` — OpenClaw
- `opencode.go` — OpenCode
- `pi.go` — Pi

**工廠函式** (`agent.go:New`)：根據 `agentType` 字串（`"claude"`, `"codex"` 等）返回對應 `Backend`。

**Session 串流模型**：

```go
type Session struct {
    Messages <-chan Message  // 執行期間串流事件（text、tool_use、thinking 等）
    Result   <-chan Result   // 完成時的最終結果
}
```

這個設計讓 daemon 可以即時將 AI 的 token/tool call 串流到 server，用戶在 UI 可以實時看到 agent 在做什麼。

---

## 任務生命週期狀態機

`agent_task_queue.status` 欄位的合法轉換：

```
queued → dispatched → running → completed
                   ↘           ↘ failed
                    cancelled   cancelled
```

**轉換觸發條件**：
- `queued`：`TaskService.EnqueueTaskForIssue()` — issue 指派給 agent 時
- `dispatched`：`ClaimNextTask()` SQL 原子操作 — daemon 認領時
- `running`：`StartTask()` — daemon 呼叫 `/tasks/{id}/start`
- `completed`：`CompleteTask()` — AI CLI 成功退出
- `failed`：`FailTask()` — AI CLI 錯誤或 blocked 狀態
- `cancelled`：用戶取消 / issue 重新指派

**租約保護**（`server/migrations/055_task_lease_and_retry.up.sql`）：
- `lease_expires_at`：防止 daemon 崩潰後任務永遠卡在 `running`
- Background sweeper `runRuntimeSweeper` 定期回收過期租約

---

## Autopilot 服務

`server/internal/service/autopilot.go` 實作兩種執行模式：

```go
// DispatchAutopilot — 核心分發邏輯
func (s *AutopilotService) DispatchAutopilot(ctx, autopilot, triggerID, source, payload)

// 兩種模式：
// "create_issue": 建立 issue + 指派給 agent（標準路徑）
// "run_only": 直接 enqueue 任務（不建立 issue）
```

**Autopilot 排程器**（`server/internal/service/cron.go`）：
- 使用 `robfig/cron/v3` 解析 cron expression
- `runAutopilotScheduler` 在 main 啟動時以 goroutine 執行
- 支援 webhook 觸發（`POST /api/autopilots/{id}/trigger`）

---

## 事件匯流排（Domain Events）

`server/internal/events/bus.go` — 進程內同步 pub/sub：

```go
type Bus struct {
    listeners      map[string][]Handler  // type-specific
    globalHandlers []Handler             // 全量監聽
}
```

**特性**：
- 同步執行（不是 async goroutine），保證事件順序
- 每個 handler 有 `recover()` 保護，單個 handler panic 不影響其他
- 事件類型定義在 `server/pkg/protocol/events.go`

**主要事件類型**（`protocol.Event*` 常數）：
- `issue:created`, `issue:updated`, `issue:deleted`
- `task:queued`, `task:dispatch`, `task:start`, `task:progress`, `task:complete`, `task:failed`
- `comment:created`, `comment:updated`
- `agent:updated`, `inbox:new`
- `autopilot:run:start`, `autopilot:run:complete`
- `chat:message:new`, `chat:message:complete`

**監聽器類型**（`server/cmd/server/router.go`）：
- `registerListeners` — WS 廣播（通用）
- `registerSubscriberListeners` — 訂閱者記錄（先於 notification，保證順序）
- `registerActivityListeners` — 活動日誌（activity_log 表）
- `registerNotificationListeners` — inbox 通知
- `registerAutopilotListeners` — autopilot 完成觸發

---

## Execution Environment（任務隔離沙箱）

`server/internal/daemon/execenv/execenv.go` — 每個任務有獨立的工作目錄：

```
~/.multica_workspaces/
  {workspace_id}/
    {task_id_short}/
      TASK.md              # 任務說明（issue title + description + criteria）
      .claude/
        skills/            # Claude Code 技能注入
          SKILL1.md
      .cursor/
        skills/            # Cursor 技能注入
      resources.json       # project resources（repo URLs 等）
      context_refs.json    # issue 的 context references
```

**GC 機制**（`server/internal/daemon/gc.go`）：
- 任務完成後記錄工作目錄元數據
- 定期 GC 清理老舊工作目錄（TTL-based）
- 特殊 GC check endpoint 讓 daemon 詢問 server 是否可以回收某個 issue 的目錄

---

## Session 恢復

這是讓 agent 在跨次任務中「記得」上一次工作的機制：

1. `task.session_id`：AI CLI 的會話 ID（Claude Code 的 `--continue <session>`）
2. `task.work_dir`：上次任務的工作目錄路徑
3. daemon 在 `CompleteTask`/`FailTask` 時回傳 sessionID + workDir
4. 下次同 (agent, issue) 的任務，daemon 從 server 取得這些值並傳入 `ExecOptions.ResumeSessionID`

**強制新 session**：`ForceFreshSession` flag（用戶點擊「重跑」時設定），daemon 跳過 resume 查詢。

---

## 前端狀態管理架構

**伺服器狀態**（TanStack Query）：
- 所有 API 資料（issues、agents、workspace 等）
- WS 事件 → `queryClient.invalidateQueries()` 觸發重新 fetch
- Key 策略：`[resource, wsId, ...params]` — workspace 切換自動失效

**客戶端狀態**（Zustand，all in `packages/core/`）：
- `authStore`：用戶資料、token、onboarding 狀態
- `chatStore`：chat session 草稿
- (Desktop only) `tabStore`：多 tab 管理，per-workspace 分組
- (Desktop only) `windowOverlayStore`：pre-workspace 流程（建立 workspace、接受邀請）

**Store 工廠模式**：所有 Zustand store 透過 factory + inject 建立，避免 singleton 測試污染：
```ts
const authStore = createAuthStore({ api, storage, onLogin, onLogout, cookieAuth })
registerAuthStore(authStore)
```

---

## Mention 系統

`server/internal/mention/` — 評論中的 agent mention：
- 解析 `@agent_name` pattern
- 觸發 `EnqueueTaskForMention(issue, agentID, triggerCommentID)`
- 允許在任何 issue（不限於已指派給該 agent）中呼叫 agent

---

## 多租戶隔離

- 所有資料表有 `workspace_id` 欄位
- middleware `RequireWorkspaceMembership` 驗證 `X-Workspace-ID` header
- Daemon token (`mdt_`) 綁定特定 workspace，只能操作自己 workspace 的 runtime/task
