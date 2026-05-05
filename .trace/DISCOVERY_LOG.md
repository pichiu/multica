# DISCOVERY_LOG.md — 探索紀錄與待解問題

> 產出日期：2026-05-05
> 覆蓋範圍：Web 搜尋、程式碼靜態分析、架構 trace（Stage 1–2）

---

## 1. Web Search 發現摘要

### 社群評價與採用狀況

- GitHub Stars：22,000+（第一週 4,000+），全球排名 #1700（[star-history](https://www.star-history.com/multica-ai/multica/)）
- 定位：2-10 人 AI-native 小團隊的任務管理平台
- 社群高度認可其「agent 作為隊友」的概念；多篇評測將其定位為 Claude Managed Agents 的開源替代品

### 競品比較

| 競品 | Multica 差異 |
|------|-------------|
| Linear | 傳統 issue tracker，無 agent 執行生命週期管理 |
| Claude Managed Agents（Anthropic 官方） | Multica 為其開源替代品，支援自托管 |
| GitHub Issues | 更基礎，無 agent runtime 概念 |

### 重要外部資源

- [官方文件](https://multica.ai/docs/)
- [How Multica Works](https://multica.ai/docs/how-multica-works)
- [Skills 文件](https://multica.ai/docs/skills)
- [Self-host Quickstart](https://multica.ai/docs/self-host-quickstart)
- [DeepWiki 自動 Wiki](https://deepwiki.com/multica-ai/multica)
- [DEV.to 深度解析](https://dev.to/truongpx396/multica-deep-dive-how-to-build-a-managed-agents-platform-54l2)
- [FAUN.dev 介紹](https://faun.pub/the-open-source-claude-managed-agents-alternative-is-here-meet-multica-7035cca69a5d)
- [Antigravity Codes 實戰指南](https://antigravity.codes/blog/multica-guide)
- [AgentConn 功能評測](https://agentconn.com/blog/multica-open-source-managed-agents-platform-review/)

### 技術重點 Takeaway

- **三元件架構**：Server（資料 hub）/ Daemon（本機執行）/ AI Coding Tool（真正的 agent）
- **Skills 系統**：遵循 Anthropic Agent Skills 開放標準，可從 ClawHub / skills.sh 匯入
- **Autopilot**：cron + webhook 觸發，每次觸發建立新 issue 並自動指派 agent
- **水平擴展**：無 Redis = 單節點 in-memory；有 Redis = 多節點 sharded stream relay

---

## 2. 既有文件 vs 程式碼落差清單

以下落差直接來自 `recon.md` 的記錄與程式碼比對：

### 落差一：`docs/docs-rewrite-plan.md` 與 `docs/docs-outline.md` 尚未落地

- 兩份草稿以簡體中文撰寫，規劃了 56 篇文件的資訊架構（6 大板塊）
- 目前 `apps/docs/` 目錄仍為初期狀態，正式文件網站尚未完成
- 程式碼功能（Autopilot、Chat、Projects、Mention 系統、session resumption）均已實作，但外部文件覆蓋率不足

### 落差二：`SELF_HOSTING_AI.md` 覆蓋不完整

- 檔案極簡，但程式碼支援 11 種 AI backend（claude、codex、copilot、opencode、openclaw、hermes、gemini、pi、cursor、kimi、kiro）
- 每種 backend 的 CLI path 設定、環境變數、模型選項均未文件化

### 落差三：68 個 Migration 無對應 Schema 文件

- `server/migrations/` 有 migration 001–067（共 68 檔）涵蓋完整資料模型
- `docs/` 下無 `schema.md` 或資料模型說明文件
- 開發者須直接讀 SQL 檔才能理解完整資料結構

### 落差四：`docs/product-overview.md` 為簡體中文且可能過時

- 截止 2026-04-21，未必反映最新功能（migrations 仍在增加中）
- README 以英文為主，但最詳盡的產品說明卻是簡體中文，讀者體驗不一致

### 落差五：Redis Relay 模式文件不足

- `SELF_HOSTING_ADVANCED.md` 提及多節點部署，但 `REALTIME_RELAY_MODE`（sharded/dual/legacy）三種模式的切換時機、風險未說明
- `dual` 模式用於滾動升級，屬於破壞性操作，需文件警示

### 落差六：PAT / Daemon Token 快取行為未說明

- `PATCache` 和 `DaemonTokenCache` 可選 Redis 支援，影響多節點認證一致性
- 目前無文件說明何時需要 Redis 支援的 token 快取

---

## 3. TODO / FIXME / HACK 彙整

### Go 端（`server/`）

| 位置 | 類型 | 說明 |
|------|------|------|
| `server/internal/middleware/workspace.go:98` | TODO | `slug → UUID` 的 lookup 可用短 TTL 快取，因 slug 為不可變欄位 |
| `server/internal/handler/label.go:93` | TODO | Labels 的字元集限制尚未實施（應排除換行符號等控制字元） |
| `server/pkg/agent/kiro.go:239` | TODO | Kiro payload 格式有兩個候選欄位，待 Kiro 官方確定 canonical 格式後刪除其中一個 |

### TypeScript 端（`packages/`、`apps/`）

靜態分析未發現 `packages/` 或 `apps/` 生產程式碼中有 `TODO`/`FIXME`/`HACK` 標記，技術債主要透過 `MUL-` issue 號碼追蹤。

### 已追蹤的歷史 Bug 修正（regression guard）

以下為程式碼中出現的 `MUL-` 參照，代表已有 regression test 保護的歷史問題：

| Issue | 說明 |
|-------|------|
| MUL-1661 | `util.ParseUUID` 靜默返回 zero UUID，導致 DELETE 成功但無實際刪除 → 已建立 UUID 解析規範 |
| MUL-1630 | daemon `poisoned.go`：AI CLI 特定輸出樣式的「毒化」偵測邏輯上限 |
| MUL-1323 / GH#1576 | 兩個 agent 陷入無限 mention loop，已注入 per-turn 提示防止 |
| MUL-1704 | repo cache 無限期卡住問題 |
| MUL-1496 / PR#1851 | screenshot URL 相關處理，影響 `cli-version.ts` |
| MUL-1198 | comment 觸發的任務在無 comment 時完成的邊界情況 |
| MUL-1397 | polling 視窗內靜默「discovery failed」 |
| MUL-975 | hostname drift 問題（daemon 重新連線後 hostname 變化） |
| MUL-820 | app sidebar workspace 切換時新 workspace 未在 dropdown 出現 |

---

## 4. 未解答的疑問

在 trace 過程中發現的模糊地帶，尚待深入確認：

1. **`apps/docs/` 的實際狀態**：`docs-rewrite-plan.md` 規劃了 Fumadocs + Next.js 文件站，但目前 `apps/docs/` 是否已有任何實際頁面，還是僅有骨架？

2. **Chat 功能的 Agent 執行路徑**：`chat_session` / `chat_message` 表已存在，但 Chat 中的 agent 對話是否走同一個 `agent_task_queue` 狀態機，還是有獨立的執行路徑？

3. **MCP（Model Context Protocol）整合現況**：社群 Issue #1351 要求 Multica 本身作為 MCP server 被 AI 操控，目前 `mcp_config` 欄位已存在於 agent 設定，但實際 MCP 支援程度（僅允許 agent 連接外部 MCP servers，還是 Multica 本身也可作為 MCP server）需要進一步確認。

4. **`workspace.settings` JSONB 欄位的完整 schema**：此欄位用於 workspace 級別功能開關，但無文件說明支援哪些 key，在不同功能的 handler 中分散使用。

5. **`EmptyClaimCache` 的 TTL 設定**：daemon 認領任務的快速路徑快取，目前不清楚 TTL 是否可調整，以及在高頻指派場景下是否存在延遲問題。

6. **GC 機制的 TTL 設定來源**：`server/internal/daemon/gc.go` 的工作目錄 TTL 是固定常數還是可設定？`session_id` 過期與工作目錄 GC 的協調機制是否存在邊界情況？

7. **Desktop app 的 Electron IPC 安全邊界**：`apps/desktop/` 使用 electron-vite，但 main process 與 renderer 之間的 IPC 邊界（preload scripts）在 trace 中未詳細覆蓋。

---

## 5. 已知技術債

直接從程式碼發現的結構性問題：

### 5.1 Daemon Legacy ID 相容層

- `server/internal/daemon/config.go` 中有 `LegacyDaemonIDs` 機制
- daemon 重新連線後 hostname 可能變化（MUL-975 已修），但相容層仍保留
- 相關函式：`LegacyDaemonIDs()`、`filterLegacyIDs()`，長期應清理

### 5.2 Markdown 格式雙重語法

- `packages/ui/markdown/file-cards.ts`：`!file[name](url)`（新語法）與 `[name](cdnUrl)`（舊語法）並存
- `packages/ui/markdown/mentions.ts`：legacy mention shortcode `[@ id="UUID" label="LABEL"]` 仍需支援
- 前端需維護兩種解析路徑，為長期負擔

### 5.3 Cookie Auth 遷移狀態

- `apps/web/components/web-providers.tsx` 仍保有 legacy localStorage token 偵測邏輯（`cookieAuth` migration）
- 新用戶走 cookie auth，舊用戶保持 localStorage token — 雙路徑增加測試和維護複雜度

### 5.4 Kiro Backend 欄位不確定性

- `server/pkg/agent/kiro.go:239`：Kiro payload 存在兩個候選欄位，等待 Kiro 官方確定
- 若 Kiro 改變 API，此處需要同步更新

### 5.5 Redis Relay `dual` 模式的滾動升級複雜度

- `DualWriteBroadcaster` 同時寫 sharded + legacy stream（雙寫），用於 rolling upgrade
- 沒有文件說明何時可以安全切回 `sharded` 模式，操作風險未明確

### 5.6 Workspace Slug Lookup 未快取

- `server/internal/middleware/workspace.go:98` 的 `TODO`：slug → UUID 每次請求都走 DB
- slug 為不可變欄位，完全適合短 TTL 快取，但尚未實施

---

## 6. 建議深入調查的區域

對開發者的後續調查建議（依優先級排列）：

### P0 — 直接影響穩定性

1. **Task lease 過期回收機制**（`server/migrations/055_task_lease_and_retry.up.sql`）
   - `runRuntimeSweeper` 掃描間隔是否足夠？`lease_expires_at` 的預設 TTL 是否合理？
   - daemon 崩潰後任務卡住的最壞等待時間？

2. **`ClaimNextTask` 原子性邊界**（`server/internal/service/task.go`）
   - 多個 daemon 並行 claim 時的 DB 鎖爭用？
   - `EmptyClaimCache` 的快取一致性（Redis vs in-memory 模式差異）？

### P1 — 功能完整性

3. **Chat + Agent 整合完整性**
   - `chat_message` 中的 agent 回覆是否有獨立的 streaming 路徑？
   - 與 issue-bound task 的狀態機是否完全分離？

4. **Autopilot webhook 觸發的認證**（`POST /api/autopilots/{id}/trigger`）
   - 外部 webhook 的認證機制（是否需要 secret token 簽名驗證）？
   - 目前是否有 rate limiting 保護？

5. **Project Resources 與 Repo 綁定的 git worktree 管理**
   - bare clone 快取的磁碟使用量上限？`~/.multica_workspaces/.repos/` 的清理策略？

### P2 — 可觀測性

6. **Prometheus metrics 覆蓋率**
   - daemon task execution time、AI CLI 的 token 消耗是否有 metrics？
   - task 失敗率 / retry 比例是否可觀測？

7. **Realtime Hub 的 client 洩漏風險**
   - WS 連線異常斷開時，`Hub` 的 client map 是否正確清理？
   - `ScopeAuthorizer` 的授權快取是否有 TTL？

---

## 7. 社群活躍議題

值得持續關注的 GitHub Issues 與外部討論：

### Issue #1351 — MCP Server 功能請求

**連結**：[Feature: MCP server for AI-native Multica orchestration from chat interfaces](https://github.com/multica-ai/multica/issues/1351)

**摘要**：社群希望 Multica 本身能作為 MCP（Model Context Protocol）server，讓 AI assistant（Claude、Cursor 等）直接透過 MCP 操控 Multica（建立 issue、查詢狀態、觸發 autopilot），而非只能透過 CLI 或 Web UI。

**目前狀態**：agent 設定中已有 `mcp_config` 欄位（允許 agent 連接**外部** MCP servers），但 Multica 本身作為 MCP server 的功能尚未實作。

**影響評估**：
- 若實作，可將 Multica 深度整合進任何支援 MCP 的 AI 工具
- 與 Autopilot 功能形成互補（Autopilot = cron/webhook 觸發，MCP = chat 觸發）
- 架構上需要新增 MCP protocol 層，並處理認證（PAT → MCP session 映射）

### 其他值得關注的技術趨勢

- **Skills 標準化**：官方文件提及遵循「Anthropic Agent Skills 開放標準」，隨 Skills 生態成熟，skill 的版本管理和相依性解析（目前僅有 `skills-lock.json`）可能成為社群需求
- **多 runtime 協調**：目前一個 workspace 可有多個 runtime（多台機器），但 task 只能指派給特定 runtime — 社群可能提出跨 runtime 負載均衡需求
- **Codex sandbox 支援**：`server/internal/daemon/execenv/codex_sandbox.go` 已有沙箱支援，但相關文件（`docs/codex-sandbox-troubleshooting.md`）表明這是相對新且不穩定的功能

---

*本文件由 Stage 1（Web 搜尋）+ Stage 2（程式碼 trace）產出，供後續文件撰寫與架構決策參考。*
