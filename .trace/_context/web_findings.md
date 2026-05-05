# Stage 1 Web 搜尋發現

## 搜尋查詢清單

1. "Multica AI managed agents platform multica.ai 2026"
2. "multica-ai/multica github open source agent task management"
3. "Multica architecture skill system autopilot agent daemon technical deep dive"
4. "multica self hosting redis websocket realtime architecture"
5. "Multica docs skills autopilot chat project features 2026"

---

## 主要資源連結

| 資源 | URL | 關鍵 takeaway |
|------|-----|---------------|
| GitHub 主庫 | https://github.com/multica-ai/multica | Apache 2.0，22,000+ stars |
| 官方網站 | https://multica.ai | Cloud 版入口 |
| 官方文件 | https://multica.ai/docs/ | 含 how-it-works、skills、self-host-quickstart |
| How Multica Works | https://multica.ai/docs/how-multica-works | 三元件架構：server / daemon / AI tool |
| Skills Docs | https://multica.ai/docs/skills | Skill = SKILL.md + 支援檔，遵循 Anthropic Agent Skills 標準 |
| Self-host Quickstart | https://multica.ai/docs/self-host-quickstart | Docker Compose 部署 |
| DeepWiki | https://deepwiki.com/multica-ai/multica | 自動生成的 codebase wiki |
| Changelog | https://multica.ai/changelog | 功能更新歷程 |

---

## 社群與評測文章

| 標題 | URL | 摘要 |
|------|-----|------|
| DEV.to 深度解析 | https://dev.to/truongpx396/multica-deep-dive-how-to-build-a-managed-agents-platform-54l2 | 架構分析，強調 agent abstraction layer 設計 |
| FAUN.dev 介紹 | https://faun.pub/the-open-source-claude-managed-agents-alternative-is-here-meet-multica-7035cca69a5d | 定位為 Claude Managed Agents 的開源替代品 |
| Antigravity Codes Guide | https://antigravity.codes/blog/multica-guide | Skills 與 Team Model 的實戰指南 |
| AgentConn Review | https://agentconn.com/blog/multica-open-source-managed-agents-platform-review/ | 功能評測 |
| AIToolly | https://aitoolly.com/ai-news/article/2026-04-12-multica-the-open-source-hosted-agent-platform-transforming-ai-into-collaborative-team-members | 新聞報導 |
| Medevel | https://medevel.com/multica-ai/ | 開源自托管評測 |
| Arun Baby Blog | https://www.arunbaby.com/ai-agents/0089-multica-agents-as-teammates/ | 使用體驗分享 |
| OpenClaw 入門指南 | https://openclawapi.org/en/blog/2026-04-11-multica-ru-men | 搭配 OpenClaw 的入門教學 |
| Star History | https://www.star-history.com/multica-ai/multica/ | #1700 全球排名 |

---

## 關鍵技術 Takeaway

### 1. 三元件架構

根據 https://multica.ai/docs/how-multica-works：
- **Server**：擁有所有資料（workspaces、issues、task queue），是 WebSocket hub
- **Daemon**：用戶本機執行，偵測 AI CLI、輪詢任務（3s 週期）、送 heartbeat（15s）
- **AI Coding Tool**：實際執行的 agent（Claude Code、Codex 等）

### 2. Skills 系統

根據 https://multica.ai/docs/skills：
- Skill = SKILL.md + 支援檔案
- 遵循 Anthropic Agent Skills 開放標準
- 可從 ClawHub 或 skills.sh 匯入
- 每個 AI tool 有自己的 skill 路徑（Claude Code 用 `.claude/skills/`，Cursor 用 `.cursor/skills/`）
- Daemon 在任務執行時自動同步 skill 到正確位置

### 3. Autopilot

- 選擇 agent + 寫 prompt + 設定 cron 表達式
- 每次觸發時，Multica 建立一個新 issue 並指派給 agent
- 也可以設定 webhook 觸發

### 4. 多節點水平擴展

- 預設：單節點，in-memory hub
- 設定 `REDIS_URL`：啟用 Redis relay，多節點 API pods 可共享 realtime 事件
- 三種 relay 模式：`sharded`（預設）、`dual`（過渡期）、`legacy`

### 5. GitHub Issue 值得關注

- Issue #1351：[Feature] MCP server for AI-native Multica orchestration from chat interfaces
  URL: https://github.com/multica-ai/multica/issues/1351
  - 表明社群希望 Multica 本身能作為 MCP server 被 AI 操控

### 6. 採用情況

- 4,000+ GitHub stars（上線第一週）
- 22,000+ stars（截至搜尋時）
- 全球排名 #1700（star-history）
- 定位：2-10 人的 AI-native 小團隊

---

## 競品與比較

- **Claude Managed Agents**（Anthropic 官方）：Multica 定位為其開源替代品
- **Linear**：傳統 issue tracker，Multica 是其 AI-native 版本
- **GitHub Issues**：更基礎，無 agent 執行生命周期管理
