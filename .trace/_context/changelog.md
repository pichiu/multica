# Stage 0 — Changelog: daf0e935..bda475c

## 摘要

- **Commits**: 79 commits
- **Files changed**: 484
- **Insertions**: 32,127 | **Deletions**: 5,758
- **Change ratio**: ~34.4% (medium) → **incremental update**

---

## Commit Log (newest → oldest)

```
bda475c refactor(reserved-slugs): single JSON source for backend + frontend (#2148)
d1a6881 docs(changelog): add v0.2.28 entry for 2026-05-08 release (#2271)
97df9b9 refactor(daemon): rename repoCache interface, relax /health test timeout (#2270)
61ce8a8 feat(daemon): add disk-usage CLI to surface per-task / per-workspace footprint (#2267)
fe8326f feat(agents): add search box to skill picker dialog (#2269)
f1dc3dc fix: keep daemon health responsive during repo lookup (#2211)
0b64f09 fix(runtimes): exclude archived agents from counts (#2166)
823f124 feat(daemon): extend GC to chat / autopilot / quick-create tasks (#2260)
b1d874e fix(timeline): rescue orphaned replies + bump page size to 50 (#2263)
eb067ff fix(server): aggregate task_usage into daily rollup table to cut DB load (#2256)
6400868 fix(timeline): off-by-one — exact-limit comments no longer triggers Show older (#2259)
bbbbcf9 fix(timeline): make Show older / Show newer affordances clearly clickable (#2257)
161194b fix(timeline): exclude activities from comment page budget (#2253)
9a3a99c fix: make CLI short IDs routable
14ab487 feat(issues): show identifier in detail page breadcrumb (#2244)
6b7294a fix(daemon): use brew prefix symlink for self-restart so Linux Cellar deletion does not orphan runtimes (#2076)
d964d37 Revert "fix(cli): add --content-file / --description-file for non-ASCII on Windows"
9650788 fix(cli): add --content-file / --description-file for non-ASCII on Windows (#2247)
00ba0aa fix(desktop): replace Electron placeholder icons with Multica asterisk for Windows + Linux (#2248)
de35656 docs(changelog): add v0.2.27 entry
47aa32a refactor(chat): unify session list into single dropdown with grouped active/archived (#2220)
a6e8ae9 fix(skills): handle GitHub API 403 / rate limit during skill import (#2215)
cc527c3 perf(heartbeat): batch runtime last_seen_at writes (#2213)
250ada1 chore(db): drop unused agent_task_queue.last_heartbeat_at (#2212)
d82a2d8 feat(skills): support importing skills from github.com URLs (#2209)
48e3131 feat: harden desktop frontend against API response drift (#2208)
dce51e3 fix(views): guard IME composition on Enter-to-submit handlers (#2207)
099dda0 fix(timeline): include merge-truncation case in has_more_before (#2204)
fe956fc feat(issues): add Copy local workdir path to issue menu (#2196)
f9cdd48 fix(projects): pre-fill status and project when creating sub-issue (#2177)
5d51a0c feat(cli): add `multica workspace update` (#2191)
d07c7c2 feat(inbox): auto-select next item after archiving the selected one (#2190)
0af67c8 fix(agent/openclaw): block tasks if openclaw < 2026.5.5 with upgrade hint (#2181)
9c00ecf fix(issues): blur sticky agent live card (#2170)
af971e1 fix(agent/openclaw): read --json from stdout, not stderr (#2101)
d0ac67d fix(skills): drop SKILL.md content from list endpoints (#2180)
53a3b33 fix(docs): keep zh internal links inside the zh locale (#2179)
c3ddb57 feat(create-issue): add border beam to switch-to-agent button (#2157)
d16c481 fix(projects): pre-fill project on per-status "+" create-issue (#2155)
11a6288 fix(timeline): legacy array shape for pre-#2128 clients (#2156)
32740d0 docs+i18n: fix terminology/runtime drift across landing, onboarding, docs (#2146)
c784a6a feat(chat): copy assistant reply + collapse process into single outer fold (#2151)
9306d60 fix(agent-live-card): self-heal stale 'is working' banner via reconcile (#2142)
4a749f1 docs(views): explain min-h-[60vh] mobile fallback in agent overview pane (#2061)
38f777d feat(autopilot): auto-pause autopilots with sustained high failure rate (#2136)
2f979ac fix(daemon): tighten quick-create prompt (#2137)
8d20a2f docs(changelog): add v0.2.26 entry for 2026-05-06 release (#2138)
e3dd31c feat(notifications): add system notifications toggle in settings (#2132)
5cf1d01 feat(settings): rename Appearance tab to Preferences and persist active tab in URL (#2131)
6d59505 fix(quick-create): remove duplicate keyboard shortcut on agent submit button (#2130)
58db751 ci(lint): enable lint in CI + fix existing lint debt (#2129)
ba14770 fix(timeline): cursor-paginated timeline to stop long-issue freeze (#2128)
3447764 feat(i18n): full rollout — 21 namespaces translated (en + zh-Hans) (#1853)
ae985ae fix(daemon): tighten 404 task-not-found semantics (#2127)
b1be9ed fix(daemon): cancel running agent when task is deleted server-side (#2107)
144661e fix(daemon/execenv): refresh stale Codex auth.json across env reuse (#2126)
0dbfbfe fix(daemon/execenv): refuse to write .gc_meta.json when issue_id is empty (#2077)
1b3c78e fix(pins): unpin missing sidebar rows (#2062)
09f0484 feat(server): redis-backed runtime liveness with DB fallback (#2121)
ee10c50 fix(daemon): trust agent's session id from session/resume across ACP backends (#2070)
140678c fix(web): redesign 404 + break NoAccessPage redirect loop (#2122)
b08594f fix(daemon): isolate runtime poll & heartbeat schedules per runtime (#2116)
a4fac51 fix(projects): add resource_count breadcrumb instead of inlining resources (#2118)
2b96733 fix(runtimes): narrow CostCell usage window from 180d to 14d (#2119)
6ef9be1 fix(chat): expose History panel + delete affordance from chat header (#2117)
60b215f feat(chat): support deleting chat sessions (#2115)
f1082b1 feat(cli): add --assignee-id / --to-id / --user-id for unambiguous targeting (#2114)
44a0ced fix(runtime): persist CLI update requests in Redis (#2113)
89b939b fix(storage): build region-qualified S3 public URLs (#2065)
8b0eeb0 fix(projects): show URL tooltip on already-attached repos (#2111)
64c605e fix(execenv): write OpenCode skills to .opencode/skills/ for native discovery (#2016)
820d575 feat(desktop): load runtime self-host config (#2012)
a7299bf refactor(projects): pass projectId prop to ProjectIssuesContent (#2110)
baac408 fix(installer): correct Windows version parsing and checksum decode (#2093)
99f6cb8 fix(projects): add New Issue button to empty project state (#2080)
b5f1e50 fix(views): split desktop/mobile sidebar state in project-detail (#2067)
00cde21 fix(views): hide archived agents from runtime detail (#2097)
1476c26 refactor(quick-create): exempt git-describe daemons from CLI gate (#2108)
9a5f5ca fix(views): coalesce repeated task_completed/task_failed activity entries (#2044)
```

---

## 變更分類

### 結構變更（新目錄 / 新模組）

| 路徑 | 類型 | 說明 |
|------|------|------|
| `packages/core/i18n/` | NEW | i18n 系統（8 個檔案）— adapter-context, browser, create-i18n, provider, react, types, user-locale-sync |
| `packages/views/i18n/` | NEW | Views i18n 整合（index, resources-types, use-t） |
| `packages/views/locales/en/` | NEW | 21 個英文 namespace JSON 檔 |
| `packages/views/locales/zh-Hans/` | NEW | 21 個簡體中文 namespace JSON 檔 |
| `server/internal/daemon/diskusage.go` | NEW | Disk usage CLI 命令 |
| `server/cmd/server/autopilot_failure_monitor.go` | NEW | Autopilot 高失敗率自動暫停監控器 |
| `server/internal/handler/heartbeat_scheduler.go` | NEW | 批量寫入 runtime last_seen_at |
| `server/internal/handler/runtime_liveness_store.go` | NEW | Redis-backed runtime liveness 快取 |
| `server/internal/handler/runtime_update_redis_store.go` | NEW | Redis-backed CLI update 請求儲存 |
| `server/internal/handler/reserved_slugs.json` | NEW | 保留 slug 清單（從 Go code 提取為 JSON） |
| `server/internal/handler/workspace_reserved_slugs.go` | NEW | 從 JSON 載入保留 slugs |
| `server/cmd/backfill_task_usage_daily/main.go` | NEW | 補填 task_usage_daily 歷史資料的一次性腳本 |
| `packages/core/api/schema.ts` / `schemas.ts` | NEW | API response schema 驗證 |

### 資料模型變更（Migrations）

| Migration | 說明 |
|-----------|------|
| 060 | 新增 `user.language` 欄位（i18n 語言設定） |
| 068 | Timeline keyset 索引（游標分頁性能） |
| 069 | 刪除 `agent_task_queue.last_heartbeat_at`（改由 heartbeat_scheduler 批量處理） |
| 072 | `task_usage.updated_at` 欄位 |
| 073 | **新表** `task_usage_daily`（token 用量日彙整，pg_cron 維護） |
| 074 | `task_usage_daily.updated_at` 索引 |
| 075 | `task_usage_daily.created_at` 索引 |
| 076 | `pg_cron` extension |
| 077 | pg_cron 任務：每小時刷新 task_usage_daily |
| 078 | `task_usage.created_at` legacy 索引 |

### API / Handler 變更

| 變更 | 說明 |
|------|------|
| skill import from GitHub URL | 新增 GitHub URL 解析 + 403 處理 |
| SKILL.md content from list endpoints | 已從技能列表 API 移除 `content` 欄位 |
| runtime liveness endpoint | Redis-backed liveness，DB fallback |
| heartbeat batching | runtime `last_seen_at` 批次寫入（效能優化） |
| chat session delete | 新增 DELETE 端點支援刪除 chat session |
| reserved-slugs | 從 Go hardcode 移至 JSON 檔（backend + frontend 共用） |
| `multica workspace update` | 新 CLI 命令（更新 workspace 設定） |
| disk-usage CLI | daemon 磁碟使用量 CLI |
| `--assignee-id` / `--to-id` | CLI 新旗標：精確指定 assignee |
| `--content-file` / `--description-file` | CLI 新旗標：Windows non-ASCII 支援 |

### Core Logic 變更

| 變更 | 說明 |
|------|------|
| Autopilot failure monitor | 高失敗率自動暫停（可設定閾值與間隔） |
| Daemon GC 擴充 | GC 現在清理 chat / autopilot / quick-create tasks（原僅清理 issue tasks） |
| Timeline cursor pagination | 游標分頁取代 offset（大 issue 不再凍結） |
| Runtime poll/heartbeat isolation | 每個 runtime 獨立排程，互不干擾 |
| execenv: OpenCode skills | OpenCode skills 寫入 `.opencode/skills/` 讓 OpenCode native discovery |
| Codex auth.json refresh | execenv 重用時強制刷新 Codex auth.json |
| Session resume 修正 | trust agent's own session_id across ACP backends |
| openclaw version gate | openclaw < 2026.5.5 → block + 升級提示 |
| Desktop: runtime self-host config | desktop app 載入 runtime 自架設定 |

### i18n

- **全新系統**：`packages/core/i18n/` — 語言切換、user preference sync、cookie adapter
- **21 個 namespace**（en + zh-Hans）：agents, auth, autopilots, chat, common, editor, inbox, invite, issues, labels, members, notifications, onboarding, projects, runtimes, settings, skills, tasks, timeline, viewer, workspace
- **user.language** DB 欄位（migration 060）

### Config / 環境變數

無新增必要環境變數。新元件行為由環境變數選配：
- Autopilot failure monitor 閾值：`MULTICA_AUTOPILOT_FAILURE_MONITOR_*`（可選）

### 前端 UI

| 變更 | 說明 |
|------|------|
| Appearance → Preferences tab | Settings tab 改名，URL 保留 tab 狀態 |
| Chat: unified session dropdown | 聊天 session 清單整合為單一 dropdown |
| Chat: delete sessions | 支援刪除 chat session |
| Desktop: Windows/Linux icons | Electron 圖示換為 Multica 星號 |
| System notifications toggle | Settings 新增系統通知開關 |
| Issue breadcrumb identifier | Issue 詳情頁顯示識別碼（如 MUL-123） |
| Copy workdir path | Issue 菜單新增複製本機工作目錄路徑 |
| Skill picker search box | 技能選擇器新增搜尋框 |
| Agent-live-card blur + reconcile | 貼近游標時模糊，定期自我修復 "is working" 旗幟 |

### Build / CI

- CI 現在執行 ESLint（`ci(lint)` #2129）

### 無影響（Docs / Locales / Tests）

- `CHANGELOG.md` — v0.2.26, v0.2.27, v0.2.28 release entries
- `packages/views/locales/**/*.json` — 翻譯字串（21 namespaces × 2 語言）
- `**/*.test.ts(x)` — 測試檔案（不影響 .trace/ 文件）
