# Elisa 架構

Elisa 是一款適合兒童使用的 IDE，它協調 AI 代理團隊來構建真實的軟體與硬體專案（Nuggets）。孩子們使用視覺化積木（Blockly）撰寫專案規格；後端將規格分解為任務有向無環圖（DAG），透過 Claude Agent SDK 執行，並即時串流回傳結果。

## 系統拓撲

```
Electron 主進程 (electron/main.ts)
  |-> 從加密儲存讀取 API 金鑰 (透過 safeStorage 使用 OS 鑰匙圈)
  |-> 選擇一個可用連接埠
  |-> 在進程中啟動 Express 伺服器
  |-> 開啟 BrowserWindow -> http://localhost:{port}

前端/ (React 19 + Vite)             後端/ (Express 5 + TypeScript)
+-----------------------+           +---------------------------+
| 積木編輯器            |  REST     | Express 伺服器            |
| (BlockCanvas)         |---------->| POST /api/sessions/:id/*  |
|                       |           |                           |
| 任務控制中心          |  WS       | 協調器 (Orchestrator)     |
| (TaskDAG, CommsFeed,  |<---------|  -> MetaPlanner (Claude)   |
|  Metrics, Deploy)     |  事件     |  -> AgentRunner (SDK)      |
|                       |           |  -> TestRunner (pytest)    |
| 底部工具列            |           |  -> GitService (simple-git)|
| (Git, Tests, Board,   |           |  -> HardwareService       |
|  Teaching)            |           |  -> TeachingEngine         |
+-----------------------+           +---------------------------+
                                              |
                                    透過 SDK query() API 執行代理
                                    依任務處理 (非同步串流)
```

在生產環境中，Express 負責處理所有請求：`/api/*` (REST)、`/ws/*` (WebSocket) 以及 `/*` (建置後的前端靜態檔案)。無需 CORS（同源）。在開發模式下，前端運行於 Vite（連接埠 5173）並代理至後端（連接埠 8000），且已啟用 CORS。

## Monorepo 佈局

```
elisa/
  package.json       根目錄 package：Electron 依賴、build/dev/dist 腳本
  electron/          Electron 主進程、preload、設定對話框
  frontend/          React SPA - 視覺化積木編輯器 + 即時儀表板
  backend/           Express 伺服器 - 協調、代理、硬體
  scripts/           建置工具 (esbuild 後端打包器)
  hardware/          ESP32 範本與共享的 MicroPython 函式庫
  docs/              產品需求文件 (elisa-prd.md)
```

根目錄 `package.json` 管理 Electron 和建置工具。前端和後端仍是獨立的 Node.js 專案，擁有各自的 `package.json`。

## 資料流：構建工作階段生命週期

```
1. 使用者在 Blockly 編輯器中排列積木
2. 點擊 GO -> blockInterpreter 將工作區轉換為 NuggetSpec JSON
3. POST /api/sessions (建立) -> POST /api/sessions/:id/start (帶著規格)
4. 後端 Orchestrator.run():
   a. 規劃 (PLAN)：  MetaPlanner 呼叫 Claude API 將規格分解為任務 DAG
   b. 執行 (EXECUTE)：串流並行池 (Promise.race，最多 3 個並行任務)
                      每個代理獲得：角色提示詞 + 任務描述 + 來自先前任務的上下文
                      代理輸出透過 SDK 串流 -> WebSocket 事件傳送至前端
                      每個任務完成後進行 Git 提交 (透過 mutex 序列化)
                      追蹤每個代理的 Token 預算；80% 時發出警告，超過時停止
   c. 測試 (TEST)：   TestRunner 執行 pytest，解析結果與覆蓋率
   d. 評閱 (REVIEW)： 選擇性的評閱代理進行審核
   e. 部署 (DEPLOY)： 將 before_deploy 規則呈現為 deploy_checklist 事件
                      如果是網頁：建置 -> 尋找服務目錄 -> 啟動本地 HTTP 伺服器 -> 開啟瀏覽器
                      如果是 ESP32：編譯 -> 燒錄 -> 序列埠監控
                      如果是 CLI 門戶：透過 CliPortalAdapter 執行 (無 shell)
5. session_complete 事件帶著摘要
```

人類閘門（Human gates）可以隨時暫停執行，需要使用者透過 REST 端點進行確認。

## 通訊協定

| 頻道 | 方向 | 用途 |
|---------|-----------|---------|
| REST | 用戶端 -> 伺服器 | 指令：建立工作階段、開始構建、閘門回應、問答回應 |
| WebSocket | 伺服器 -> 用戶端 | 事件：任務進度、代理輸出、測試結果、教學時刻、錯誤 |

WebSocket 路徑：`/ws/session/:sessionId`

在開發模式下，Vite（連接埠 5173）將 `/api/*` 和 `/ws/*` 代理至後端（連接埠 8000）。在生產環境（Electron）中，Express 在同一連接埠上靜態提供前端檔案 —— 不需要代理。

## 核心抽象

### NuggetSpec
由 blockInterpreter 從 Blockly 工作區產生。驅動整個流程。包含：目標、需求、風格、代理、硬體配置、部署目標、工作流標記、技能、規則。

### 任務 DAG
具有依賴關係的任務有向無環圖。由 MetaPlanner 產出。由 Orchestrator 按拓撲順序執行。使用 Kahn 演算法（`utils/dag.ts`）。

### 構建工作階段 (Build Session)
單次執行運行的記憶體狀態。追蹤：工作階段 ID、階段、任務、代理、提交、事件、教學時刻、測試結果、Token 使用量。無資料庫 —— 內容全部存在記憶體中。

### 代理角色
- **Builder**: 撰寫原始碼
- **Tester**: 撰寫並執行測試
- **Reviewer**: 評閱程式碼品質
- **Custom**: 使用者定義的角色

各代理透過 Claude Agent SDK 的 `query()` API 執行，注入來自 `backend/src/prompts/` 的角色特定系統提示詞。

## 狀態機

```
閒置 (idle) -> 規劃 (planning) -> 執行 (executing) -> 測試 (testing) -> 評閱 (reviewing) -> 部署 (deploying) -> 完成 (done)
                                    ^                                               |
                               人類閘門 (透過 REST 暫停/恢復)                     繼續工作
                                                                                   |
                                                                                   v
                                                                             設計 (反覆構建)
```

## 關鍵模式

- **事件驅動 UI**：前端所有狀態更新皆透過 WebSocket 事件處理器處理。無輪詢。
- **代理隔離**：每個代理任務作為獨立的 SDK `query()` 呼叫運行。代理之間除了透過寫入工作區 `.elisa/` 的上下文摘要外，不共享狀態。
- **上下文鏈**：每個任務完成後，摘要會寫入 `.elisa/context/nugget_context.md`。後續代理接收此檔案作為輸入，建立上下文鏈。
- **優雅降級**：缺少工具（git, pytest, mpremote, serialport）會觸發警告而非當機。
- **Bearer Token 身分驗證**：伺服器啟動時產生隨機驗證 Token。所有 `/api/*` 路由（`/api/health` 除外）皆需要 `Authorization: Bearer <token>`。WebSocket 升級需要 `?token=<token>` 查詢參數。在 Electron 中，Token 透過 IPC 分享給渲染進程。
- **內容安全**：所有代理提示詞均包含內容安全章節，強制產出適合兒童的內容（8-14 歲）。使用者控制的預留位置值在提示詞揷入前會進行過濾。
- **中止傳播**：Orchestrator 的 AbortController 信號會轉發至每個代理的 SDK `query()` 呼叫。取消或發生錯誤時，代理會立即中止。
- **API 金鑰管理**：開發中從 `ANTHROPIC_API_KEY` 環境變數讀取。Electron 中透過 OS 鑰匙圈 (`safeStorage`) 加密並存儲於本地。子進程（測試執行器、燒錄腳本、建置）接收不含 API 金鑰的過濾環境變數。

## 儲存

- **工作階段狀態**：記憶體中的 `Map<sessionId, Session>`，可選擇 JSON 永久化以進行當機恢復
- **工作區**：每個工作階段的暫存目錄（`/tmp/elisa-nugget-{timestamp}`）或使用者選擇的目錄。包含產生的程式碼、測試、git 儲存庫、`.elisa/` 中繼資料以及設計產出（nugget.json, dag.json, workspace.json 等）
- **localStorage**：工作區 JSON、技能與規則在瀏覽器中自動儲存 (`elisa:workspace`, `elisa:skills`, `elisa:rules`)。頁面載入時恢復。
- **Nugget 檔案**：用於匯出/匯入的 `.elisa` zip 格式（工作區 + 技能 + 規則 + 產生的程式碼）
- **無資料庫**

## 硬體路徑

透過 serialport 函式庫支援 ESP32：
1. 透過 USB VID:PID 偵測開發板 (Heltec LoRa, ESP32-S3, CH9102)
2. 透過 `py_compile` 編譯 MicroPython
3. 透過 `mpremote` 燒錄
4. 序列埠監控速度為 115200 baud，透過 WebSocket 串流至前端

## 模組級文件

各子系統的深入背景資訊存放在各目錄下的 CLAUDE.md 檔案中：
- `frontend/CLAUDE.md` - 前端架構、元件樹、狀態管理
- `backend/CLAUDE.md` - 後端架構、服務、API 介面
- `backend/src/services/CLAUDE.md` - 服務職責與互動
- `frontend/src/components/CLAUDE.md` - 元件階層與模式

## Electron 打包

Elisa 以 Electron 桌面應用程式形式分發。建置流程：
1. `npm run build:frontend` -- Vite 將 React SPA 建置到 `frontend/dist/`
2. `npm run build:backend` -- esbuild 將 Express 伺服器打包到 `backend/dist/server-entry.js`
3. `npm run build:electron` -- tsc 編譯 `electron/main.ts` 與 `preload.ts`
4. `npm run dist` -- electron-builder 打包成安裝程式 (Windows 為 NSIS, macOS 為 DMG)

開發模式（在根目錄執行 `npm run dev`）：同時執行後端、前端與 Electron。Electron 載入 `http://localhost:5173` (Vite HMR)。生產環境：Electron 載入 `http://localhost:{可用連接埠}`，Express 於此提供所有內容。
