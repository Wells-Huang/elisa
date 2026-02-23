<p align="center">
  <img src="frontend/assets/Elisa.png" alt="Elisa" width="200" />
</p>

<h1 align="center">Elisa</h1>

<p align="center">
  一個基於積木的視覺化程式設計環境，孩子們（以及大人）可以透過組合積木來設計軟體，接著由 AI 代理將其實作出來。
</p>

<p align="center">
  <img alt="授權：MIT" src="https://img.shields.io/badge/license-MIT-blue.svg" />
  <img alt="React 19" src="https://img.shields.io/badge/React-19-61dafb.svg" />
  <img alt="TypeScript" src="https://img.shields.io/badge/TypeScript-5.9-3178c6.svg" />
  <img alt="Express 5" src="https://img.shields.io/badge/Express-5-000000.svg" />
  <img alt="CI" src="https://github.com/zoidbergclawd/elisa/actions/workflows/ci.yml/badge.svg" />
</p>

---

<p align="center">
  <img src="frontend/assets/elisa.gif" alt="Elisa UI 演示" width="800" />
</p>

## 什麼是 Elisa？

Elisa 是一款將積木式視覺化程式設計轉化為真實運行軟體的教育工具。使用者透過拖放積木來描述他們的需求 —— 目標、功能、限制、視覺風格、硬體目標 —— 然後一個 AI 代理團隊會協作將其實作。整個過程是透明的：你可以即時觀察代理進行規劃、撰寫程式碼、測試及部署。

## 快速開始

```bash
# 後端 (終端機 1)
cd backend && npm install && npm run dev    # localhost:8000

# 前端 (終端機 2)
cd frontend && npm install && npm run dev   # localhost:5173
```

需要 Node.js 20+ 以及 Anthropic API 金鑰 (`ANTHROPIC_API_KEY` 環境變數)。已在 Windows 和 macOS 上測試。完整設定請參閱 [入門指南](docs/getting-started.md)。

## 功能特點

**基於積木的設計** —— 橫跨 9 個類別組合積木來描述您的專案：目標、需求、風格、代理、流程控制、硬體、部署、技能與規則。無需撰寫程式碼。

**AI 代理協調** —— 元規劃器 (Meta-planner) 將您的規格分解為任務有向無環圖 (DAG)。構建者、測試者、評閱者以及自定義代理依據依賴順序執行任務，並包含重試與代理間通訊機制。

**即時建置透明度** —— 透過三欄式佈局觀察建置過程：積木編輯器（左側）、包含任務圖與代理通訊的任務控制中心（右側），以及包含 git 時間軸、測試結果、序列埠輸出與教學時刻的底部工具列。

**硬體整合** —— 直接針對 ESP32 開發板進行開發。提供 LED、按鈕、感測器、LoRa、蜂鳴器與計時器的積木。具備自動偵測、編譯並透過 USB 燒錄的功能。

**人機協作 (Human-in-the-Loop)** —— 可在任何位置插入「與我確認」閘門。代理會暫停並在繼續之前尋求批准。建置過程中可回答代理的問題。

**教學時刻** —— 教學引擎會在代理工作時提供適合年齡的程式設計概念解釋，將每一次建置轉化為學習過程。

**技能與規則** —— 建立可重複使用的提示詞片段（技能）以及基於觸發條件的規則，以塑造代理在不同建置中的行為。

## 代理驅動的規格驅動開發 (Agentic Spec-Driven Development)

Elisa 是一款用於 **代理驅動規格驅動開發** 的 IDE —— 這種規範中，軟體創作者關心的是 **規格與測試**，而非程式碼。使用者透過視覺化積木定義 *想要什麼*；AI 代理則負責找出 *如何實作*。程式碼是產出的產物，類似於編譯後的二進位檔案。

IDE 中的每個積木都對應到真實的 AI 工程原語：

| Elisa 原語 | 積木範例 | AI 工程概念 |
|---|---|---|
| **目標 (Goal)** | `nugget_goal`, `nugget_template` | 頂層系統提示詞。驅動整個代理流程的根指令。 |
| **需求 (Requirement)** | `feature`, `constraint`, `when_then`, `has_data` | 提示詞限制與驗收標準。由元規劃器分解為任務 DAG。 |
| **技能 (Skill)** | `use_skill` | 注入代理上下文的可重複提示詞片段。具有分支的多步驟流程（詢問使用者、執行代理、調用另一個技能）。 |
| **規則 (Rule)** | `use_rule` | 代理行為的 Pre/post 鉤子。在事件（如 `before_commit` 或 `on_test_fail`）觸發時觸發的限制條件。 |
| **門戶 (Portal)** | `portal_tell`, `portal_when`, `portal_ask` | 工具使用：MCP 伺服器、CLI 指令、硬體序列埠 I/O。門戶是代理的手 —— 接觸外部世界的方式。 |
| **僕從 (Minion)** | `agent_builder`, `agent_tester`, `agent_reviewer`, `agent_custom` | 具有獨立系統提示詞、工具存取權限與 Token 預算的專門代理角色。自定義僕從讓使用者定義新角色。 |
| **流程 (Flow)** | `first_then`, `at_same_time`, `check_with_me` | DAG 依賴關係、並行執行與人機協作閘門。控制協調拓撲。 |
| **部署 (Deploy)** | `deploy_web`, `deploy_esp32`, `deploy_both` | 部署目標。網頁部署至 Cloud Run；硬體編譯並透過 USB 燒錄 MicroPython。 |

視覺化積木工作區會編譯為 **NuggetSpec** —— 這是一個經過 Zod 驗證且作為唯一事實來源的 JSON 結構。協調器採用該規格，元規劃器將其分解為任務，代理依此執行。使用者迭代規格並重新執行；程式碼隨之重新產生。

## 架構

```
瀏覽器 (React SPA)  <──REST + WebSocket──>  Express 伺服器
      |                                           |
   積木編輯器                                協調器管線
   任務控制中心                          元規劃器 -> 代理執行器
   底部工具列                           Git 服務, 測試執行器
                                        硬體服務, 教學引擎
```

每個代理作為獨立的 SDK `query()` 呼叫運行。無資料庫 —— 所有工作階段狀態皆存在記憶體中。完整系統設計請參閱 [ARCHITECTURE.md](ARCHITECTURE.md)。

## 專案結構

```
elisa/
  frontend/          React + Vite + Blockly SPA
    src/components/
      BlockCanvas/   積木編輯器 + 積木定義 + 解釋器
      MissionControl/ 任務 DAG、代理通訊饋送、指標
      BottomBar/     Git 時間軸、測試結果、開發板輸出、教學
      shared/        執行按鈕、彈窗、提示、頭像
      Skills/        技能與規則 CRUD 編輯器
  backend/           Express + WebSocket 伺服器
    src/services/
      orchestrator   建置管線控制器
      agentRunner    Claude Agent SDK 執行器
      metaPlanner    NuggetSpec -> 任務 DAG 分解
      gitService     各工作階段的 git 儲存庫管理
      testRunner     pytest 執行與覆蓋率解析
      hardwareService ESP32 偵測/編譯/燒錄/序列埠
      teachingEngine 概念課程與去重
  hardware/          適用於 ESP32 的 MicroPython 範本
  docs/              詳細文件
```

## 文件

- [使用者手冊](docs/manual/README.md) —— Elisa 使用完整指南
- [入門指南](docs/getting-started.md) —— 前提條件、安裝、首次建置
- [API 參考](docs/api-reference.md) —— REST 端點、WebSocket 事件、NuggetSpec 結構
- [積木參考](docs/block-reference.md) —— 完整積木調色盤指南
- [前端 README](frontend/README.md) —— 前端架構與開發指南
- [後端 README](backend/README.md) —— 後端架構與開發指南
- [架構](ARCHITECTURE.md) —— 系統級設計概覽

## 授權

[MIT](LICENSE) -- (c) 2026 Zoidberg
