# Stage 1/2 — 變更分類 + 更新計畫

## 分類結果

| 類別 | 檔案數 | 比例 |
|------|--------|------|
| i18n（locales/*.json 翻譯字串） | ~90 | 18.6% |
| 測試 | ~30 | 6.2% |
| 文件（CHANGELOG、comments） | ~15 | 3.1% |
| Core logic 變更 | ~150 | 31% |
| Data model（migrations + generated） | ~25 | 5.2% |
| Frontend UI | ~100 | 20.7% |
| Build / CI | ~10 | 2.1% |
| 其他 | ~64 | 13.2% |

**結論**：34.4% 中量變更 → **增量更新**（非完整重新 trace）

---

## 各文件影響評估

### INDEX.md — 需更新

| 項目 | 變更 |
|------|------|
| Tech stack 表 | 新增 i18n（next-intl / react-i18next 等）、pg_cron |
| 指令速查 | 新增 `multica workspace update` CLI 命令 |
| 術語表 | 新增 i18n 相關術語 |

### ARCHITECTURE.md — 需更新

| 項目 | 變更 |
|------|------|
| 架構圖（mermaid） | 新增 i18n 模組到 Shared Packages；autopilot_failure_monitor 到 Server |
| 元件說明 | heartbeat_scheduler、runtime_liveness_store 說明 |
| 設計決策 | 新增 DD-7：i18n 架構；DD-8：timeline cursor pagination |

### DATA_MODEL.md — 需更新

| 項目 | 變更 |
|------|------|
| user 表 | 新增 `language` 欄位（migration 060） |
| agent_task_queue 表 | 移除 `last_heartbeat_at`（migration 069） |
| 新表 task_usage_daily | migration 073（bucket_date, workspace_id, runtime_id, provider, model, token counts） |
| Migration 歷程 | 更新到 migration 078 |

### API_SURFACE.md — 輕量更新

| 項目 | 變更 |
|------|------|
| Skills API | 新增 GitHub URL import；list 端點移除 content 欄位 |
| Chat API | 新增 DELETE /chat/sessions/:id |
| CLI 章節 | 新增 workspace update 命令 |

### CODEBASE_MAP.md — 需更新

| 項目 | 變更 |
|------|------|
| 目錄樹 | 新增 packages/core/i18n/、packages/views/i18n/、packages/views/locales/ |
| Server 目錄 | 新增 heartbeat_scheduler.go、runtime_liveness_store.go、autopilot_failure_monitor.go |
| Daemon 目錄 | 新增 diskusage.go |
| "我想改 X" 查表 | 新增 i18n 相關條目 |

### DEV_GUIDE.md — 輕量更新

| 項目 | 變更 |
|------|------|
| CLI 參考 | 新增 multica workspace update 命令；新增 --assignee-id 等旗標 |
| 新增 i18n 小節 | 說明如何新增翻譯 |

### DISCOVERY_LOG.md — 需更新

| 項目 | 變更 |
|------|------|
| 新增「增量更新記錄」章節 | 記錄 v0.2.26～v0.2.28 重大變更 |
| 更新技術債 | 移除已解決的技術債（timeline cursor pagination 已修） |

---

## 更新策略

- **DATA_MODEL.md**：patch 寫入（新增欄位 / 新增表格 / 更新 migration 歷程）
- **ARCHITECTURE.md**：patch 寫入（更新 mermaid 圖 + 新增設計決策）
- **INDEX.md**：patch 寫入（新增 tech stack 列 + 更新指令 + 術語表）
- **CODEBASE_MAP.md**：patch 寫入（新增目錄條目）
- **API_SURFACE.md**：patch 寫入（新增端點）
- **DEV_GUIDE.md**：patch 寫入（新增 i18n 章節 + CLI 參考更新）
- **DISCOVERY_LOG.md**：patch 寫入（新增增量更新章節）
