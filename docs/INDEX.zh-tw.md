# Elisa 專案索引

這是一個基於積木的視覺化程式設計 IDE，孩子們透過組合積木來協調 AI 代理團隊，進而構建軟體。

## 目錄導覽

| 路徑 | 說明 |
|------|-------------|
| `electron/` | Electron 主進程、preload、設定對話框 |
| `backend/` | Express 5 API 伺服器 + 協調引擎 |
| `backend/src/routes/` | REST 端點處理程序 (sessions, hardware, skills, workspace) |
| `backend/src/services/` | 核心服務：協調器、執行器、硬體、門戶 |
| `backend/src/services/phases/` | 管線階段處理程序：規劃、執行、測試、部署 |
| `backend/src/models/` | TypeScript 類型定義 (session, skillPlan) |
| `backend/src/prompts/` | 代理角色提示詞 + 課程範本 |
| `backend/src/utils/` | DAG、驗證、日誌、Token、上下文、超時工具 |
| `backend/src/tests/` | 後端測試 |
| `backend/src/tests/behavioral/` | 服務與路由的整合/行為測試 |
| `backend/src/tests/fixtures/` | 測試固定裝置資料 (計畫、規格) |
| `frontend/` | React 19 + Vite SPA |
| `frontend/src/components/` | UI 元件樹 |
| `frontend/src/components/BlockCanvas/` | Blockly 編輯器、積木定義、解釋器 |
| `frontend/src/components/AgentTeam/` | 代理卡片 + 通訊饋送 |
| `frontend/src/components/TaskMap/` | DAG 視覺化 (@xyflow/react) |
| `frontend/src/components/MissionControl/` | 共享組件：TaskDAG, CommsFeed, 指標面板 |
| `frontend/src/components/BottomBar/` | 分頁：時間軸、測試、開發板、學習、進度、Token |
| `frontend/src/components/Skills/` | 技能 CRUD 彈窗 + 範本庫 |
| `frontend/src/components/Rules/` | 規則 CRUD 彈窗 + 範本庫 |
| `frontend/src/components/Portals/` | 門戶連接彈窗 |
| `frontend/src/components/shared/` | 可重複使用組件：標籤、按鈕、彈窗、提示、頭像 |
| `frontend/src/hooks/` | React hooks (工作階段狀態、健康檢查、WebSocket、開發板偵測、技能) |
| `frontend/src/lib/` | 工具函式 (nugget 檔案、技能範本、術語) |
| `frontend/src/types/` | TypeScript 定義 |
| `hardware/` | MicroPython ESP32 範本 + 共享函式庫 |
| `hardware/lib/` | 共享的 MicroPython 函式庫 (`elisa_hardware.py`) |
| `hardware/templates/` | ESP32 專案範本 (blink, LoRa) |
| `scripts/` | 建置工具 (esbuild 後端打包器、連接埠清理器、子目錄安裝器) |
| `docs/` | 產品與技術文件 |
| `support/` | ESP32 韌體二進位檔案 |

## 文件導覽

| 檔案 | 範圍 | 內容 |
|------|-------|----------|
| `ARCHITECTURE.md` | 系統 | 拓撲、資料流、狀態機、模組分解 |
| `.claude/CLAUDE.md` | 專案 | 技術棧、環境變數、開發設定、規範 |
| `backend/CLAUDE.md` | 模組 | 服務、API 介面、階段處理程序、關鍵模式 |
| `backend/README.md` | 開發 | 指令、結構、如何新增端點/角色 |
| `backend/src/services/CLAUDE.md` | 模組 | 協調器、階段、執行器、互動模式 |
| `frontend/CLAUDE.md` | 模組 | 元件、狀態、Blockly 整合、WS 協定 |
| `frontend/README.md` | 開發 | 指令、元件結構、如何新增積木 |
| `frontend/src/components/CLAUDE.md` | 模組 | 元件階層、BlockCanvas 子系統、UI 模式 |
| `docs/INDEX.md` | 專案 | 本檔案。結構、文件與關鍵檔案的主索引 |
| `docs/manual/README.md` | 使用者 | 完整使用者手冊：工作區、積木、建置、技能、規則、門戶、硬體 |
| `docs/getting-started.md` | 使用者 | 安裝、快速入門、首次建置、疑難排解 |
| `docs/api-reference.md` | API | REST 端點、WebSocket 事件、NuggetSpec 結構 |
| `docs/block-reference.md` | 使用者 | 積木類別及其說明 |
| `docs/elisa-prd.md` | 產品 | 產品需求文件：願景、功能、目標受眾 |

## 關鍵原始碼檔案

### 路由

| 檔案 | 角色 |
|------|------|
| `backend/src/routes/sessions.ts` | /api/sessions/* 端點 (建立、開始、停止、閘門、提問、匯出) |
| `backend/src/routes/hardware.ts` | /api/hardware/* 端點 (偵測、燒錄) |
| `backend/src/routes/skills.ts` | /api/skills/* 端點 (執行、回答、列表) |
| `backend/src/routes/workspace.ts` | /api/workspace/* 端點 (儲存、讀取設計檔案) |

### 服務

| 檔案 | 角色 |
|------|------|
| `backend/src/services/orchestrator.ts` | 精簡的協調器：規劃 -> 執行 -> 測試 -> 部署 |
| `backend/src/services/metaPlanner.ts` | 透過 Claude API 將 NuggetSpec 分解為任務 DAG |
| `backend/src/services/agentRunner.ts` | 透過 Claude Agent SDK `query()` 執行帶串流輸出的代理 |
| `backend/src/services/sessionStore.ts` | 具備 JSON 持久化功能的工作階段狀態管理 |
| `backend/src/services/gitService.ts` | 各工作階段的 git 初始化及基於任務的提交 |
| `backend/src/services/hardwareService.ts` | ESP32 偵測、MicroPython 編譯、燒錄、序列埠監控 |
| `backend/src/services/testRunner.ts` | 具備覆蓋率解析功能的 pytest / Node 測試執行器 |
| `backend/src/services/skillRunner.ts` | 帶有使用者互動的逐步 SkillPlan 執行 |
| `backend/src/services/teachingEngine.ts` | 適合年齡的學習時刻 (課程 + Claude) |
| `backend/src/services/portalService.ts` | 具備指令允許清單的 MCP + CLI 門戶轉接器 |
| `backend/src/services/narratorService.ts` | 為建置事件產生講述者訊息 (Claude Haiku) |
| `backend/src/services/permissionPolicy.ts` | 根據策略規則自動解析代理權限請求 |

### 階段

| 檔案 | 角色 |
|------|------|
| `backend/src/services/phases/planPhase.ts` | MetaPlanner 調用、DAG 設定、初期教學時刻 |
| `backend/src/services/phases/executePhase.ts` | 串流並行任務執行 (3 個並行，Promise.race) |
| `backend/src/services/phases/testPhase.ts` | 測試執行器調用與結果回報 |
| `backend/src/services/phases/deployPhase.ts` | 網頁預覽、硬體燒錄、門戶部署、序列埠監控 |
| `backend/src/services/phases/types.ts` | 共享的 PhaseContext 與 SendEvent 類型 |

### 工具

| 檔案 | 角色 |
|------|------|
| `backend/src/utils/dag.ts` | 具備 Kahn 拓撲排序、循環偵測功能的任務 DAG |
| `backend/src/utils/specValidator.ts` | NuggetSpec 的 Zod schema 驗證 |
| `backend/src/utils/contextManager.ts` | 檔案清單、nugget 上下文、結構化摘要 |
| `backend/src/utils/sessionLogger.ts` | 各工作階段的結構化日誌，存於 `.elisa/logs/` |
| `backend/src/utils/sessionPersistence.ts` | 原子 JSON 檢查點/恢復 |
| `backend/src/utils/tokenTracker.ts` | Token 追蹤、各代理成本、預算強制執行 |
| `backend/src/utils/withTimeout.ts` | 支援 AbortSignal 的通用 Promise 超時封裝 |
| `backend/src/utils/constants.ts` | 超時、限制、間隔、預設模型的命名常量 |
| `backend/src/utils/pathValidator.ts` | 工作區路徑驗證 (系統/敏感目錄黑名單) |
| `backend/src/utils/safeEnv.ts` | 過濾後的 process.env 副本 (移除 ANTHROPIC_API_KEY) |
| `backend/src/utils/findFreePort.ts` | 從起始連接埠掃描可用的 TCP 連接埠 |
| `backend/src/utils/anthropicClient.ts` | Anthropic SDK 用戶端的單例工廠 |

### 提示詞

| 檔案 | 角色 |
|------|------|
| `backend/src/prompts/metaPlanner.ts` | 任務分解的系統提示詞 |
| `backend/src/prompts/builderAgent.ts` | 構建者角色提示詞範本 |
| `backend/src/prompts/testerAgent.ts` | 測試者角色提示詞範本 |
| `backend/src/prompts/reviewerAgent.ts` | 評閱者角色提示詞範本 |
| `backend/src/prompts/narratorAgent.ts` | 建置事件旁白的講述者角色提示詞 |
| `backend/src/prompts/teaching.ts` | 教學時刻課程與範本 |

### 組件

| 檔案 | 角色 |
|------|------|
| `frontend/src/App.tsx` | 根組件、佈局、彈窗路由 |
| `frontend/src/components/BlockCanvas/BlockCanvas.tsx` | Blockly 編輯器封裝 |
| `frontend/src/components/BlockCanvas/blockDefinitions.ts` | 自訂積木類型 (10 個類別) |
| `frontend/src/components/BlockCanvas/blockInterpreter.ts` | 工作區 -> NuggetSpec JSON 轉換 |
| `frontend/src/components/BlockCanvas/toolbox.ts` | Blockly 側欄類別 |
| `frontend/src/components/BlockCanvas/skillFlowBlocks.ts` | 技能流程積木定義 |
| `frontend/src/components/BlockCanvas/skillInterpreter.ts` | 技能流程工作區 -> SkillPlan 轉換 |
| `frontend/src/components/AgentTeam/AgentTeamPanel.tsx` | 全寬代理卡片 + 通訊饋送 |
| `frontend/src/components/TaskMap/TaskMapPanel.tsx` | 全寬互動式任務 DAG |
| `frontend/src/components/shared/MinionAvatar.tsx` | 旁白/僕從角色的動畫頭像 |
| `frontend/src/components/MissionControl/MissionControlPanel.tsx` | 包含旁白饋送 + 僕從小隊的主任務控制佈局 |
| `frontend/src/components/MissionControl/MinionSquadPanel.tsx` | 帶有狀態勳章和任務分配的僕從小隊卡片 |
| `frontend/src/components/MissionControl/NarratorFeed.tsx` | 具備情緒指標的滾動式旁白訊息饋送 |

### Hooks

| 檔案 | 角色 |
|------|------|
| `frontend/src/hooks/useBuildSession.ts` | 所有工作階段狀態 + WebSocket 事件分發 |
| `frontend/src/hooks/useSkillSession.ts` | 獨立技能執行狀態 + WebSocket 事件 |
| `frontend/src/hooks/useBoardDetect.ts` | ESP32 開發板偵測輪詢 |
| `frontend/src/hooks/useHealthCheck.ts` | 後端就緒狀態輪詢 |
| `frontend/src/hooks/useWebSocket.ts` | 具備自動重新連線功能的 WebSocket 連線 |

### 函式庫

| 檔案 | 角色 |
|------|------|
| `frontend/src/lib/nuggetFile.ts` | .elisa nugget 檔案儲存/讀取 (基於 JSZip) |
| `frontend/src/lib/skillTemplates.ts` | 內建技能與規則範本 |
| `frontend/src/lib/terminology.ts` | 適合兒童的術語映射 (技術術語 -> 親切的標籤) |

### Electron

| 檔案 | 角色 |
|------|------|
| `electron/main.ts` | 視窗建立、後端生命週期、API 金鑰儲存 |
| `electron/preload.ts` | 渲染進程的上下文橋接 |

## 資料流

```
Blockly 工作區
  -> blockInterpreter -> NuggetSpec JSON (經 Zod 驗證)
  -> REST POST /api/sessions/:id/start
  -> orchestrator -> metaPlanner -> 任務 DAG
  -> executePhase (透過 Promise.race 執行 3 個並行代理)
  -> 代理輸出透過 SDK 串流 -> WebSocket 事件
  -> useBuildSession -> React UI 狀態更新
  -> (選用) 「繼續工作」 -> 設計階段 -> 使用現有工作區 + git 歷程重新建置
```
