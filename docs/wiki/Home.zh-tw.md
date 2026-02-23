# Elisa

**這是一款專為兒童設計的 IDE，孩子們可以協調 AI 代理團隊來構建真實的軟體與硬體專案。**

Elisa 是一款受 Scratch 啟發的視覺化程式設計工具。孩子們可以組合積木來描述他們想要構建的內容 —— 包含目標、功能、風格、代理團隊以及部署目標。在後台，這些積木會驅動 AI 代理（由 Claude 提供支持）來規劃任務、撰寫程式碼、執行測試並部署成果。產出的是真實、可運行的軟體：一個可以分享的網站，或是燒錄到 ESP32 微控制器的程式碼。

Elisa 同時是 **教學工具** 也是 **實作工具**。在每一個步驟中，它都會使用適合兒童的語言解釋工程概念 —— 如原始碼控制、測試、分解、程式碼評閱 —— 並將這些知識與建置過程交織在一起。

## 關鍵功能

- **視覺化積木編輯器** — 透過拖放積木 (Blockly) 來組合專案規格。共有 10 個類別：目標、需求、風格、技能、規則、門戶、僕從、流程、部署以及技能流程。
- **AI 代理團隊** — 配置具備個性的構建者、測試者、評閱者及自定義「僕從」代理。最多可同時讓 3 位代理平行工作。
- **即時任務控制中心** — 即時觀察代理規劃任務、撰寫程式碼並部署。包含互動式任務圖 (DAG)、講述者饋送以及僕從狀態卡片。
- **技能與規則** — 建立可重複使用的提示詞片段（技能）與護欄（規則）來塑造代理行為。複合技能可以使用視覺化流程編輯器來鏈接多個步驟。
- **門戶 (Portals)** — 透過 MCP 伺服器、CLI 工具或序列埠/USB 裝置連接外部工具與硬體。
- **硬體整合** — 透過 USB 偵測、燒錄並監控 ESP32 開發板。內建 MicroPython 編譯與序列埠監控功能。
- **教學層** — 情境式的學習時刻，在 Git 提交、測試與分解發生時同步解釋其概念。
- **儲存與分享** — 自動儲存到瀏覽器、儲存到資料夾，或匯出為 `.elisa` nugget 檔案。隨附範例專案供快速開始。

## 快速開始

```bash
git clone https://github.com/zoidbergclawd/elisa.git
cd elisa
npm install
npm run dev:electron
```

詳見 [入門指南 (Getting-Started)](Getting-Started) 獲取完整安裝步驟。

## 使用者指南

| 頁面 | 說明 |
|------|-------------|
| [入門指南 (Getting Started)](Getting-Started) | 前提條件、安裝、首次啟動、第一次建置 |
| [工作區編輯器 (Workspace Editor)](Workspace-Editor) | 畫布、調色盤、側欄、分頁標籤 |
| [積木參考 (Block Reference)](Block-Reference) | 所有 10 個積木類別的欄位與行為說明 |
| [建置 Nugget (Building a Nugget)](Building-a-Nugget) | GO 按鈕、建置階段、任務控制中心、人類閘門 |
| [技能 (Skills)](Skills) | 建立、編輯、複合技能、範本、上下文變數 |
| [規則 (Rules)](Rules) | 建立、觸發條件、範本 |
| [門戶 (Portals)](Portals) | MCP/CLI/序列埠、範本、能力詳細說明 |
| [硬體整合 (Hardware Integration)](Hardware-Integration) | ESP32 設定、偵測、燒錄、序列埠監控 |
| [儲存與讀取 (Saving and Loading)](Saving-and-Loading) | 自動儲存、.elisa 檔案、工作區存取、範例 |
| [疑難排解 (Troubleshooting)](Troubleshooting) | 就緒勳章、API 金鑰、連線、建置錯誤 |

## 開發者指南

| 頁面 | 說明 |
|------|-------------|
| [架構 (Architecture)](Architecture) | 系統拓撲、Monorepo 佈局、資料流、狀態機 |
| [API 參考 (API Reference)](API-Reference) | REST 端點、WebSocket 事件、NuggetSpec 結構 |
| [開發指南 (Development Guide) ](Development-Guide) | 開發設定、npm 指令碼、測試、如何新增端點/積木/代理 |
| [代理系統 (Agent System)](Agent-System) | 代理角色、提示詞、執行模型、上下文鏈、權限 |

## 技術棧 (Tech Stack)

| 層級 | 技術 |
|-------|-------------|
| 桌面端 | Electron 35, electron-builder, electron-store + safeStorage |
| 前端 | React 19, Vite 7, TypeScript 5.9, Tailwind CSS 4, Blockly 12 |
| 後端 | Express 5, TypeScript 5.9, ws 8, Zod 4, Claude Agent SDK |
| 硬體 | ESP32 上的 MicroPython (透過 serialport + mpremote) |
| 測試 | Vitest + Testing Library |

## 相關連結

- [GitHub 儲存庫](https://github.com/zoidbergclawd/elisa)
- [產品需求文件 (PRD)](https://github.com/zoidbergclawd/elisa/blob/main/docs/elisa-prd.md)
