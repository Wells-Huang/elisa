# 後端模組

使用 Express 5 + TypeScript 的伺服器。透過 Claude Agent SDK 協調 AI 代理團隊，並透過 WebSocket 將結果串流至前端。

## 技術棧

- Express 5, TypeScript 5.9, Node.js (ES 模組)
- ws 8 (WebSocket), simple-git 3, serialport 12, @anthropic-ai/sdk, @anthropic-ai/claude-agent-sdk
- Zod 4 (NuggetSpec 驗證)
- archiver 7 (用於 nugget 匯出的 zip 串流)
- Vitest (測試)

## 結構

```
src/
  server.ts              精簡的組合根 (Composition Root)。掛載路由模組、WS 伺服器，並匯出 startServer()。
  routes/
    sessions.ts          /api/sessions/* 端點 (建立、開始、停止、閘門、提問、匯出)
    hardware.ts          /api/hardware/* 端點 (偵測、燒錄)
    skills.ts            /api/skills/* 端點 (執行、回答、列表)
    workspace.ts         /api/workspace/* 端點 (儲存、讀取設計檔案)
  models/
    session.ts           類型定義：Session, Task, Agent, BuildPhase, WSEvent
  services/
    orchestrator.ts      精簡的協調器：依序委派給各階段處理程序
    sessionStore.ts      整合後的工作階段狀態 (取代 4 個並行 Map)
    phases/
      types.ts           共享的 PhaseContext 與 SendEvent 類型
      planPhase.ts       MetaPlanner 調用、DAG 設定
      executePhase.ts    任務執行迴圈 (並行、git 互斥鎖、上下文鏈)
      testPhase.ts       測試執行器調用、結果回報
      deployPhase.ts     硬體燒錄、門戶部署
    agentRunner.ts       透過 SDK query() API 執行代理並串流輸出
    metaPlanner.ts       呼叫 Claude API 將 NuggetSpec 分解為任務 DAG
    gitService.ts        Git 初始化、每個任務的提交、差異追蹤
    hardwareService.ts   ESP32 偵測、編譯、燒錄、序列埠監控
    testRunner.ts        執行 Python 的 pytest 或 JS 的 Node 測試執行器。解析結果與覆蓋率。
    skillRunner.ts       逐步執行 SkillPlans (詢問使用者、分支、執行代理、調用技能)
    teachingEngine.ts    產生情境學習時刻 (課程 + API 備案)
    narratorService.ts   為建置事件產生旁白訊息 (Claude Haiku)
    permissionPolicy.ts  根據策略規則自動解析代理權限請求
  prompts/
    metaPlanner.ts       任務分解的系統提示詞
    builderAgent.ts      構建者角色提示詞範本
    testerAgent.ts       測試者角色提示詞範本
    reviewerAgent.ts     評閱者角色提示詞範本
    teaching.ts          教學時刻課程與範本
    narratorAgent.ts     建置事件旁白的講述者角色提示詞
  utils/
    dag.ts               具備 Kahn 拓撲排序、循環偵測功能的任務 DAG
    contextManager.ts    建置檔案清單、nugget 上下文、結構化摘要、狀態快照
    specValidator.ts     用於 NuggetSpec 驗證的 Zod schema (字串上限、陣列限制)
    sessionLogger.ts     各工作階段的結構化日誌，存於 .elisa/logs/
    sessionPersistence.ts 工作階段檢查點/恢復的原子 JSON 持久化
    tokenTracker.ts      追蹤輸入/輸出 Token、各代理成本及預算限制
    withTimeout.ts       支援 AbortSignal 的通用 Promise 超時封裝
    constants.ts         超時、限制、間隔、預設模型的命名常量
    pathValidator.ts     工作區路徑驗證 (系統/敏感目錄黑名單)
    safeEnv.ts           過濾後的 process.env 副本 (移除 ANTHROPIC_API_KEY)
    findFreePort.ts      從起始連接埠掃描可用的 TCP 連接埠
    anthropicClient.ts   Anthropic SDK 用戶端的單例工廠
```

## API 介面

### REST 端點
| 方法 | 路徑 | 用途 |
|--------|------|---------|
| GET | /api/health | 就緒檢查 (API 金鑰 + SDK 狀態) |
| POST | /api/sessions | 建立工作階段 |
| POST | /api/sessions/:id/start | 帶著 NuggetSpec 開始構建 |
| POST | /api/sessions/:id/stop | 取消構建 |
| POST | /api/sessions/:id/gate | 人類閘門回應 |
| POST | /api/sessions/:id/question | 回答代理提問 |
| GET | /api/sessions/:id | 工作階段狀態 |
| GET | /api/sessions/:id/tasks | 任務清單 |
| GET | /api/sessions/:id/git | 提交記錄 |
| GET | /api/sessions/:id/tests | 測試結果 |
| GET | /api/sessions/:id/export | 將 nugget 目錄匯出為 zip |
| POST | /api/workspace/save | 將設計檔案儲存至工作區目錄 |
| POST | /api/workspace/load | 從工作區目錄讀取設計檔案 |
| POST | /api/skills/run | 開始獨立的技能執行 |
| POST | /api/skills/:id/answer | 回答技能提問 |
| GET | /api/hardware/detect | 偵測 ESP32 (僅快速 VID:PID) |
| POST | /api/hardware/flash/:id | 燒錄至開發板 |

### WebSocket 事件 (伺服器 -> 用戶端)
`planning_started`, `plan_ready`, `task_started`, `task_completed`, `task_failed`, `agent_output`, `commit_created`, `token_usage`, `budget_warning`, `test_result`, `coverage_update`, `deploy_started`, `deploy_progress`, `deploy_checklist`, `deploy_complete` (網頁部署包含 `url?`), `serial_data`, `human_gate`, `user_question`, `skill_*`, `teaching_moment`, `narrator_message`, `permission_auto_resolved`, `minion_state_change`, `workspace_created`, `error`, `session_complete`

## 關鍵模式

- **工作階段狀態**：記憶體中的 Map，具備用於檢查點/恢復的可選 JSON 持久化。5 分鐘寬限期後自動清理。
- **NuggetSpec 驗證**：Zod schema 在 `/api/sessions/:id/start` 進行驗證 (字串上限、陣列限制、門戶指令允許清單)。
- **每個任務進行 SDK 查詢**：每個代理任務使用 `@anthropic-ai/claude-agent-sdk` 呼叫 `query()`，並設定 `permissionMode: 'bypassPermissions'`。預設 `maxTurns=25` (`MAX_TURNS_DEFAULT`)。重試時每次增加 10 個輪次 (`MAX_TURNS_RETRY_INCREMENT`)，重試進度為：25 → 35 → 45。
- **過時中繼資料清理**：每次建置時，`setupWorkspace()` 在重新建立之前會移除先前工作階段的 `.elisa/{comms,context,status}`。保留 `.elisa/logs/`、原始碼檔案與 `.git/`。
- **結構化摘要注入**：代理任務提示詞包含從工作區原始碼檔案中提取的函式/類別簽名 (透過 `ContextManager.buildStructuralDigest()`)，讓代理無需讀取每個檔案即可定位。
- **重試上下文**：失敗的任務會重試（最多 2 次），並在提示詞前加上「重試嘗試」標頭，指示代理跳過定位並直接進入實作。
- **串流並行執行**：透過 Promise.race 池同時運行最多 3 個獨立任務。一旦有任務完成即安排新任務。Git 提交透過互斥鎖序列化。
- **Token 預算**：每個工作階段預設為 50 萬 Token 限制。80% 時發出警告事件。超過時優雅停止。追蹤每個代理的成本。
- **上下文鏈**：每個任務完成後，摘要 + 結構化摘要會寫入 `.elisa/context/nugget_context.md`。
- **取消**：透過 AbortController 進行 `Orchestrator.cancel()`；信號會傳播到 Agent SDK 的 `query()` 呼叫。發生錯誤時工作階段狀態設為 `done`。
- **內容安全**：所有代理提示詞均強制執行適合年齡的產出 (8-14 歲)。預留位置值在揷入前會進行過濾 (`sanitizePlaceholder()`)。
- **燒錄互斥鎖**：`HardwareService.flash()` 透過 Promise 鏈互斥鎖將並行呼叫序列化。
- **優雅關機**：SIGTERM/SIGINT 處理程序會取消協調器、關閉 WS 伺服器、10 秒強制結束。`SessionStore.onCleanup` 調用 `ConnectionManager.cleanup()` 進行 WS 解除。
- **優雅降級**：缺少外部工具 (git, pytest, mpremote) 會產生警告，而非當機。
- **超時設定**：代理 = 300 秒，測試 = 120 秒，燒錄 = 60 秒。任務重試限制 = 2。

## 伺服器模式

`server.ts` 匯出 `startServer(port, staticDir?)` 供 Electron 使用。當獨立運行 (`tsx src/server.ts`) 時，它會自動偵測直接執行並在 `process.env.PORT` (預設 8000) 啟動。

- **開發模式** (無 `staticDir`)：為 `CORS_ORIGIN` 環境變數 (預設 `http://localhost:5173`) 啟用 CORS。前端由 Vite 獨立提供。
- **生產模式** (具備 `staticDir`)：無 CORS。Express 提供前端靜態檔案 + SPA 備案 (非 API 路由指向 `index.html`)。

## 配置

- `PORT`：後端連接埠 (預設 8000)，或由 Electron 選擇可用連接埠
- `CORS_ORIGIN`：覆寫開發模式下的 CORS 來源 (預設 `http://localhost:5173`)
- `CLAUDE_MODEL`：覆寫代理與教學引擎的模型 (預設 `claude-opus-4-6`)
- `ANTHROPIC_API_KEY`：Claude API/SDK 存取所需
- Claude 模型：可透過 `CLAUDE_MODEL` 環境變數配置 (預設 claude-opus-4-6)
