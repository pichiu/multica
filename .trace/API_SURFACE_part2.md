# API_SURFACE_part2.md — Multica API 參考（Part 2/2）

> 接續 [API_SURFACE.md](./API_SURFACE.md)

---

## 目錄

4. [關鍵 API 詳情（後半）](#4-關鍵-api-詳情後半)
5. [WebSocket 協議](#5-websocket-協議)
6. [Daemon API](#6-daemon-api)
7. [Error Handling](#7-error-handling)

---

## 4. 關鍵 API 詳情（後半）

### 4.1 POST /api/daemon/runtimes/{runtimeId}/tasks/claim

Daemon 原子認領下一個可執行任務。

**Auth：** `Authorization: Bearer mdt_<token>`

**Request**
```json
POST /api/daemon/runtimes/<runtime-uuid>/tasks/claim
Authorization: Bearer mdt_<token>
Content-Type: application/json

{}
```

**Response** `200 OK`（有任務可認領）
```json
{
  "task": {
    "id": "<task-uuid>",
    "issue_id": "<issue-uuid>",
    "agent_id": "<agent-uuid>",
    "runtime_id": "<runtime-uuid>",
    "status": "dispatched",
    "type": "issue",
    "prompt": "<task prompt text>",
    "skills": ["<skill SKILL.md content>"],
    "repos": [{ "url": "https://github.com/org/repo" }],
    "session_id": "<previous-claude-session-id>",
    "created_at": "2026-05-05T10:00:00Z"
  }
}
```

**Response** `200 OK`（無任務）
```json
{ "task": null }
```

備註：`session_id` 非空時，Claude Code 可用 `--resume` 恢復前一次 session。

---

### 4.2 POST /api/chat/sessions/{id}/messages

向 Chat Session 發送使用者訊息，並觸發 agent 任務。

**Request**
```json
POST /api/chat/sessions/<session-uuid>/messages
X-Workspace-ID: <workspace-uuid>
Content-Type: application/json

{ "content": "請分析這段程式碼的效能瓶頸" }
```

**Response** `201 Created`
```json
{
  "message_id": "<message-uuid>",
  "task_id": "<task-uuid>",
  "created_at": "2026-05-05T10:00:00Z"
}
```

後續透過 WebSocket 接收 `chat:message`（streaming 片段）與 `chat:done`（完成）事件。

---

## 5. WebSocket 協議

### 5.1 連線

```
GET /ws
Upgrade: websocket

# Cookie Auth（前端）
Cookie: token=<jwt>

# PAT Auth（程式/CLI）
Authorization: Bearer mul_<token>

# Workspace 過濾（Query param，可選）
?workspace_id=<uuid>
?workspace_slug=<slug>
```

伺服器驗證 `Origin` 是否在 `CORS_ALLOWED_ORIGINS` 白名單內。

### 5.2 WS 事件完整列表

所有事件格式：
```json
{ "type": "<event-type>", "payload": { ... } }
```

#### Issue 事件

| 事件 | 觸發時機 | Payload |
|------|---------|---------|
| `issue:created` | 新 issue 建立 | 完整 `IssueResponse` |
| `issue:updated` | issue 欄位更新 | 部分 `IssueResponse`（不含 labels） |
| `issue:deleted` | issue 刪除 | `{ id }` |
| `issue_labels:changed` | labels 異動 | `{ issue_id, labels[] }` |

#### Task 事件

| 事件 | 觸發時機 | 狀態轉換 |
|------|---------|---------|
| `task:queued` | 任務入隊 | `∅ → queued` |
| `task:dispatch` | Daemon 認領 | `queued → dispatched` |
| `task:progress` | Agent 進度更新 | payload：`{ summary, step, total }` |
| `task:message` | Agent 訊息片段 | streaming 內容 |
| `task:completed` | 任務完成 | `running → completed` |
| `task:failed` | 任務失敗 | `running → failed` |
| `task:cancelled` | 任務取消 | 任意狀態 → cancelled |

#### Chat 事件

| 事件 | 觸發時機 |
|------|---------|
| `chat:message` | Agent 回應訊息（streaming） |
| `chat:done` | Chat session 回應完成 |
| `chat:session_read` | 用戶標記 session 已讀 |

#### Inbox 事件

| 事件 | 觸發時機 |
|------|---------|
| `inbox:new` | 新收件匣項目 |
| `inbox:read` | 單項目標為已讀 |
| `inbox:archived` | 單項目封存 |
| `inbox:batch-read` | 批次已讀 |
| `inbox:batch-archived` | 批次封存 |

#### Agent / Workspace 事件

| 事件 | 觸發時機 |
|------|---------|
| `agent:status` | Agent 狀態變更 |
| `agent:created` | 新 agent 建立 |
| `agent:archived` | Agent 封存 |
| `agent:restored` | Agent 還原 |
| `workspace:updated` | Workspace 資料更新 |
| `workspace:deleted` | Workspace 刪除 |
| `member:added/updated/removed` | 成員異動 |

#### 其他事件

| 事件 | 觸發時機 |
|------|---------|
| `comment:created/updated/deleted` | 評論 CRUD |
| `reaction:added/removed` | 評論 reaction |
| `issue_reaction:added/removed` | Issue reaction |
| `skill:created/updated/deleted` | Skill CRUD |
| `project:created/updated/deleted` | Project CRUD |
| `project_resource:created/deleted` | Project resource 異動 |
| `label:created/updated/deleted` | Label CRUD |
| `autopilot:created/updated/deleted` | Autopilot CRUD |
| `autopilot:run_start/run_done` | Autopilot 執行 |
| `invitation:created/accepted/declined/revoked` | 邀請事件 |
| `pin:created/deleted/reordered` | Pin 事件 |
| `activity:created` | Activity log |
| `subscriber:added/removed` | Issue 訂閱者異動 |

#### Daemon WS 專用事件

| 事件 | 方向 | 說明 |
|------|------|------|
| `daemon:register` | Server → Daemon | 確認 daemon 已完成註冊 |
| `daemon:heartbeat` | Daemon → Server | Daemon 心跳信號 |
| `daemon:heartbeat_ack` | Server → Daemon | Server 確認心跳，回傳 task available 狀態 |
| `daemon:task_available` | Server → Daemon | 有新任務可認領（wakeup 通知） |

---

## 6. Daemon API

所有路由前綴：`/api/daemon/`  
**Auth：** Daemon Token（`mdt_`）優先；亦接受 User JWT / PAT（具 workspace 成員身份）。

### 6.1 Daemon 生命週期

| Method | Path | 說明 |
|--------|------|------|
| `POST` | `/api/daemon/register` | 向 server 註冊 daemon 及其 runtimes |
| `POST` | `/api/daemon/deregister` | 取消註冊 |
| `POST` | `/api/daemon/heartbeat` | 定期心跳（預設每 15 秒），含 task claim 邏輯 |
| `GET` | `/api/daemon/ws` | Daemon 專用 WebSocket（接收 wakeup 通知） |
| `GET` | `/api/daemon/workspaces/{workspaceId}/repos` | 取得 workspace 關聯 git repos |

**DaemonRegisterRequest：**
```json
{
  "workspace_id": "<workspace-uuid>",
  "daemon_id": "<persistent-uuid>",
  "legacy_daemon_ids": ["<old-uuid>"],
  "device_name": "MacBook-Pro",
  "cli_version": "0.2.0",
  "launched_by": "desktop",
  "runtimes": [
    {
      "name": "claude-code",
      "type": "claude",
      "version": "1.2.3",
      "status": "online"
    }
  ]
}
```

### 6.2 任務執行生命週期

| Method | Path | 說明 |
|--------|------|------|
| `POST` | `/api/daemon/runtimes/{runtimeId}/tasks/claim` | 原子認領任務 |
| `GET` | `/api/daemon/runtimes/{runtimeId}/tasks/pending` | 查詢 pending 任務清單 |
| `GET` | `/api/daemon/tasks/{taskId}/status` | 取得任務狀態 |
| `POST` | `/api/daemon/tasks/{taskId}/start` | 標記任務開始執行 |
| `POST` | `/api/daemon/tasks/{taskId}/progress` | 回報執行進度 |
| `POST` | `/api/daemon/tasks/{taskId}/complete` | 回報完成 |
| `POST` | `/api/daemon/tasks/{taskId}/fail` | 回報失敗 |
| `POST` | `/api/daemon/tasks/{taskId}/messages` | 回報訊息片段（streaming） |
| `GET` | `/api/daemon/tasks/{taskId}/messages` | 列出任務訊息 |
| `POST` | `/api/daemon/tasks/{taskId}/usage` | 回報 token 用量 |
| `POST` | `/api/daemon/tasks/{taskId}/session` | 記錄 Claude session ID |
| `POST` | `/api/daemon/runtimes/{runtimeId}/recover-orphans` | 重新領取遺棄任務 |

**TaskProgressRequest：**
```json
{ "summary": "Running tests...", "step": 3, "total": 5 }
```

**TaskCompleteRequest：**
```json
{
  "pr_url": "https://github.com/org/repo/pull/123",
  "output": "All tests pass. Created PR #123.",
  "session_id": "<claude-session-uuid>",
  "work_dir": "/home/user/.multica/workspaces/repo"
}
```

**TaskFailRequest（⚠️ 未驗證完整結構）：**
```json
{ "error": "exit status 1: compilation failed" }
```

### 6.3 Runtime 管理（User 側）

| Method | Path | 說明 |
|--------|------|------|
| `GET` | `/api/runtimes` | 列出 workspace 的所有 runtimes |
| `DELETE` | `/api/runtimes/{runtimeId}` | 刪除 runtime 記錄 |
| `POST` | `/api/runtimes/{runtimeId}/update` | 觸發 CLI 更新請求 |
| `GET` | `/api/runtimes/{runtimeId}/update/{updateId}` | 查詢更新請求狀態 |
| `POST` | `/api/runtimes/{runtimeId}/models` | 請求列出可用 AI models |
| `GET` | `/api/runtimes/{runtimeId}/models/{requestId}` | 查詢 model list 請求 |
| `POST` | `/api/runtimes/{runtimeId}/local-skills` | 請求列出本地 skills |
| `POST` | `/api/runtimes/{runtimeId}/local-skills/import` | 請求匯入本地 skill |
| `GET` | `/api/runtimes/{runtimeId}/usage` | 取得 runtime 用量統計 |
| `GET` | `/api/runtimes/{runtimeId}/usage/by-agent` | 用量（依 agent 分組） |
| `GET` | `/api/runtimes/{runtimeId}/usage/by-hour` | 用量（依小時分組） |
| `GET` | `/api/runtimes/{runtimeId}/activity` | 取得 runtime 任務活動記錄 |

### 6.4 Daemon 行為變更紀錄

| 行為 | 說明 | PR |
|------|------|-----|
| **任務刪除時取消 agent** | Daemon 偵測到 server 端任務被刪除後，立即取消正在執行的 agent | #2107 |
| **每個 runtime 獨立 poll & heartbeat 排程** | Daemon 不再共用全域定時器；每個 runtime 有獨立的 poll 與 heartbeat 週期，避免跨 runtime 干擾 | #2116 |
| **404 task-not-found 語意收緊** | 任務輪詢回傳 404 時，Daemon 視為任務已消失並停止相關 agent，不再靜默重試 | #2127 |

---

## 7. Error Handling

### 7.1 Error Response 格式

所有錯誤均返回 JSON，`Content-Type: application/json`：

```json
{ "error": "human-readable error message" }
```

### 7.2 常見 HTTP 狀態碼

| 狀態碼 | 情境 |
|--------|------|
| `400 Bad Request` | 請求格式錯誤、必填欄位缺失、UUID 格式無效 |
| `401 Unauthorized` | 未提供 token 或 token 無效/過期 |
| `403 Forbidden` | 已認證但無權限（如非 admin 執行 admin 操作） |
| `404 Not Found` | 資源不存在或不屬於當前 workspace |
| `409 Conflict` | 唯一性衝突（如 workspace slug 重複） |
| `500 Internal Server Error` | Server 內部錯誤（DB 故障、panic 被 Recoverer 捕捉） |
| `503 Service Unavailable` | 第三方服務未配置（如 Google OAuth 未設定） |

### 7.3 UUID 驗證規則

後端 handler 嚴格區分兩種 UUID 解析方式（Issue #1661 修復）：

| 來源 | 方法 | 失敗行為 |
|------|------|---------|
| 來自用戶輸入（URL 參數、request body） | `parseUUIDOrBadRequest` | 返回 `400 Bad Request` |
| 來自 DB 的 UUID round-trip（sqlc 返回值） | `parseUUID`（panic variant） | panic → `500`（由 `middleware.Recoverer` 處理） |

### 7.4 Issue ID 解析

`/api/issues/{id}` 的 `{id}` 支援兩種格式：

1. **UUID**：標準 UUID 格式
2. **Human-readable Identifier**：`PREFIX-NUMBER` 格式（如 `MUL-42`）— server 自動解析

---

*最後更新：2026-05-09*  
*來源：`server/cmd/server/router.go`、`server/pkg/protocol/events.go`、`server/internal/handler/`*
