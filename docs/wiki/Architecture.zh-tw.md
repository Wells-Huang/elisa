# 架構 (Architecture)

Elisa 是一款適合兒童使用的 IDE，它協調 AI 代理團隊來構建真實的軟體與硬體專案。孩子們使用視覺化積木 (Blockly) 撰寫專案規格；後端將規格分解為任務有向無環圖 (DAG)，透過 Claude Agent SDK 執行，並即時將結果以串流方式回傳。

## 系統拓撲

```
Electron 主進程 (electron/main.ts)
  |→ 從加密儲存讀取 API 金鑰 (透過 safeStorage 使用 OS 鑰匙圈)
  |→ 選擇一個可用連接埠
  |→ 在進程中啟動 Express 伺服器
  |→ 開啟 BrowserWindow → http://localhost:{port}

前端/ (React 19 + Vite)             後端/ (Express 5 + TypeScript)
+-----------------------+           +---------------------------+
| 積木編輯器            |  REST     | Express 伺服器            |
| (BlockCanvas)         |---------->| POST /api/sessions/:id/*  |
|                       |           |                           |
| 任務控制中心          |  WS       | 協調器 (Orchestrator)     |
| (TaskDAG, CommsFeed,  |<---------| → MetaPlanner (Claude)    |
|  Metrics, Deploy)     |  事件     | → AgentRunner (SDK)       |
|                       |           | → TestRunner (pytest)     |
| 底部工具列            |           | → GitService (simple-git) |
| (Git, Tests, Board,   |           | → HardwareService         |
|  Teaching)            |           | → TeachingEngine          |
+-----------------------+           +---------------------------+
                                             |
                                   透過 SDK query() API 執行代理
                                   個別任務處理 (非同步串流)
```

在生產環境中，Express 負責處理所有事務：`/api/*` (REST)、`/ws/*` (WebSocket) 以及 `/*` (前端構建後的靜態檔案)。這使得不需要 CORS（同源）。在開發模式下，前端運行於 Vite (連接埠 5173) 並代理至後端 (連接埠 8000)，且會啟用 CORS。

## Monorepo 佈局

```
elisa/
  package.json       根專案套件：Electron 相依性、建置/開發/發布指令碼
  electron/          Electron 主進程、preload、設定對話框
  frontend/          React SPA — 視覺化積木編輯器 + 即時儀表板
  backend/           Express 伺服器 — 協調、代理、硬體
  scripts/           建置工具 (esbuild 後端打包器)
  hardware/          ESP32 範本與共享的 MicroPython 函式庫
  docs/              文件 (手冊、API 參考、PRD)
```

根目錄的 `package.json` 管理 Electron 與建置工具。前端與後端保持為獨立的 Node.js 專案，擁有各自的 `package.json`。

## 資料流：建置工作階段生命週期

```
1. 使用者在 Blockly 編輯器中排列積木
2. 點擊 GO → blockInterpreter 將工作區轉換為 NuggetSpec JSON
3. POST /api/sessions (建立) → POST /api/sessions/:id/start (帶著規格)
4. 後端 Orchestrator.run():
   a. 規劃 (PLAN)：元規劃器呼叫 Claude API 將規格分解為任務 DAG
   b. 執行 (EXECUTE)：串流並行執行池 (Promise.race，最多 3 個並行任務)
                    每位代理獲得：角色提示詞 + 任務說明 + 前序任務上下文
                    代理輸出透過 SDK 串流傳輸 → WebSocket 事件傳至前端
                    完成每項任務後進行 Git 提交 (透過互斥鎖序列化)
                    追蹤每位代理的 Token 預算；80% 時發出警告，超過時停止
   c. 測試 (TEST)：測試執行器執行 pytest，解析結果與覆蓋率
   d. 評閱 (REVIEW)：選用的評閱者代理階段
   e. 部署 (DEPLOY)：將「部署前 (before_deploy)」規則呈現為 deploy_checklist 事件
                    若為網頁：建置 → 尋找服務目錄 → 啟動本機 HTTP 伺服器 → 開啟瀏覽器
                    若為 ESP32：編譯 → 燒錄 → 序列埠監控
                    若為 CLI 門戶：透過 CliPortalAdapter 執行 (不經過 shell)
5. 傳送包含摘要的 session_complete 事件
```

人類閘門 (Human gates) 可以在任何時間點暫停執行，需要使用者透過 REST 端點批准。

## 通訊協定

| 頻道 | 方向 | 用途 |
|---------|-----------|---------|
| REST | 用戶端 → 伺服器 | 指令：建立工作階段、開始建置、閘門回應、提問回答 |
| WebSocket | 伺服器 → 用戶端 | 事件：任務進度、代理輸出、測試結果、教學時刻、錯誤 |

WebSocket 路徑：`/ws/session/:sessionId`

## 核心抽象

### NuggetSpec
由積木解釋器從 Blockly 工作區產出的 JSON 結構。驅動整個管線。包含：目標、需求、風格、代理、硬體配置、部署目標、工作流旗標、技能、規則。由伺服器端透過 Zod schema 驗證。

### 任務 DAG (Task DAG)
具備相依關係的任務有向無環圖。由元規劃器產生。由協調器按拓撲順序執行。使用 Kahn 演算法 (`backend/src/utils/dag.ts`)。

### 建置工作階段 (Build Session)
單次執行運行的記憶體中狀態。追蹤：工作階段 ID、階段、任務、代理、提交、事件、教學時刻、測試結果、Token 使用量。無資料庫 —— 所有內容均存在記憶體中，並提供選用的 JSON 持久化功能以進行當機恢復。

### 代理角色 (Agent Roles)
- **構建者 (Builder)** — 撰寫原始碼
- **測試者 (Tester)** — 撰寫並執行測試
- **評閱者 (Reviewer)** — 評審程式碼品質
- **自定義 (Custom)** — 使用者定義的個性

每位代理透過 Claude Agent SDK 的 `query()` API 執行，並由 `backend/src/prompts/` 注入角色特定的系統提示詞。

## 狀態機

```
閒置 (idle) → 規劃 (planning) → 執行 (executing) → 測試 (testing) → 評閱 (reviewing) → 部署 (deploying) → 完成 (done)
                      ^                                                                         |
                  人類閘門 (透過 REST 暫停/恢復)                                               繼續改進
                                                                                                |
                                                                                                v
                                                                                              設計 (design)
```

## 關鍵模式

- **事件驅動 UI**：前端所有狀態更新皆透過 WebSocket 事件處理程序流動。不使用輪詢。
- **代理隔離**：每個代理任務作為獨立的 SDK `query()` 調用運行。代理之間除了透過上下文摘要外，不共享狀態。
- **上下文鏈**：每項任務結束後，摘要會寫入 `.elisa/context/nugget_context.md`。後續代理會將此作為輸入。
- **優雅降級**：遺失工具 (git, pytest, mpremote, serialport) 會導致警告，而非系統當機。
- **Bearer 權杖授權**：伺服器啟動時產生隨機授權權杖。所有 `/api/*` 路由（除了 `/api/health`）皆需 `Authorization: Bearer <token>` 請求頭。WebSocket 升級需要 `?token=<token>`。
- **內容安全**：所有代理提示詞皆包含「內容安全」區塊，強制執行適合 8–14 歲年齡的產出。使用者控制的佔位符值在提示詞揷值前會先經過清理。
- **中止傳播**：協調器的 AbortController 信號會轉發至每位代理的 SDK `query()` 調用中。
- **API 金鑰管理**：在開發環境，從 `ANTHROPIC_API_KEY` 環境變數讀取。在 Electron 下，透過 OS 鑰匙圈 (`safeStorage`) 加密。子進程接收到的是移除 API 金鑰後的清理後環境變數。

## 儲存

- **工作階段狀態**：記憶體中的 `Map<sessionId, Session>`，具備選用的 JSON 持久化供當機恢復。
- **工作區**：每個工作階段使用暫存目錄，或由使用者選擇的目錄。包含產出的程式碼、測試、Git 儲存庫、`.elisa/` 中繼資料。
- **localStorage**：工作區 JSON、技能及規則在瀏覽器中自動儲存。頁面載入時恢復。
- **Nugget 檔案**：用於匯出/匯入的 `.elisa` zip 格式（工作區 + 技能 + 規則 + 產出的程式碼）。
- **無資料庫**。

## Electron 打包

1. `npm run build:frontend` — Vite 將 React SPA 建置至 `frontend/dist/`
2. `npm run build:backend` — esbuild 將 Express 伺服器打包至 `backend/dist/server-entry.js`
3. `npm run build:electron` — tsc 編譯 `electron/main.ts` 與 `preload.ts`
4. `npm run dist` — electron-builder 打包成安裝程式 (Windows 使用 NSIS, macOS 使用 DMG)

## 模組級文件

各個子系統的更深入背景資訊位於各目錄下的 CLAUDE.md 檔案中：
- `frontend/CLAUDE.md` — 前端架構、元件樹、狀態管理
- `backend/CLAUDE.md` — 後端架構、服務、API 介面
- `backend/src/services/CLAUDE.md` — 服務職責與互動
- `frontend/src/components/CLAUDE.md` — 元件階層與模式
