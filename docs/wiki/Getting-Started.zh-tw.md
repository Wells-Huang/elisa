# 入門指南 (Getting Started)

## 前提條件

| 需求 | 備註 |
|-------------|-------|
| Node.js 20+ | 必要。可從 [nodejs.org](https://nodejs.org) 下載 |
| Anthropic API 金鑰 | 必要。請至 [console.anthropic.com](https://console.anthropic.com) 獲取 |
| Python 3.10+ | 選填。僅當您的建置包含測試任務時需要 |
| ESP32 + mpremote | 選填。僅硬體專案需要。可使用 `pip install mpremote` 安裝 |

## 安裝

### Electron 應用程式 (推薦)

```bash
git clone https://github.com/zoidbergclawd/elisa.git
cd elisa
npm install
npm run dev:electron
```

這會安裝所有相依性（根目錄、後端及前端）並啟動 Electron 整合環境，包含後端、前端及桌面視窗。

> **重要：** 請務必使用 `npm run dev:electron`。不要只使用 `npm run dev` —— 後者僅會在瀏覽器中啟動後端 + 前端，而不具備 Electron 桌面視窗功能。

### 僅限瀏覽器 (開發模式)

僅當您明確需要不使用 Electron 運行時才使用此方法：

```bash
npm install
npm run dev    # 後端 (連接埠 8000) + 前端 (連接埠 5173)
```

在瀏覽器中開啟 `http://localhost:5173`。

## 設定您的 API 金鑰

- **Electron 應用程式**：首次啟動時會跳出設定對話框要求輸入。您的金鑰會透過作業系統鑰匙圈 (safeStorage) 進行加密並安全存儲。
- **開發模式**：啟動後端前，請先設定 `ANTHROPIC_API_KEY` 環境變數。

右上角的就緒勳章會顯示連線狀態：

| 勳章 (Badge) | 顏色 | 代表意義 |
|-------|-------|---------|
| 就緒 (Ready) | 綠色 | 所有項目皆已配置且運作正常 |
| 檢查中 (Checking...) | 灰色 | 正在測試連線 |
| 需要 API 金鑰 | 黃色 | 未設定 API 金鑰，或金鑰無效 |
| 離線 (Offline) | 紅色 | 無法連線至後端伺服器 |

## 首次啟動介面說明

主畫面分為三個區域：

- **標題列 (Header)** — 包含 Elisa 標誌、分頁標籤（工作區 / 任務控制中心）、GO 按鈕及就緒勳章。
- **主編輯區** — 積木編輯畫布，左側為調色盤，右側為側欄。
- **底部列** — 包含時間軸、測試、開發板、學習、進度及 Token 標籤。這些會在建置過程中顯示資訊。

當您第一次啟動時，會自動開啟「範例選擇器 (Example Picker)」，供您從預設專案開始。

## 您的第一次建置

1. **設計** — 從調色盤將積木拖曳到畫布上。最少需要一個包含描述的「目標」積木。或者從範例選擇器中選取一個範例。
2. **按下 GO** — 標題列的綠色按鈕會將您的設計發送給 AI 代理。
3. **選擇資料夾** — Elisa 會詢問產出的專案檔案要存放在何處。
4. **觀察** — 視圖會切換到任務控制中心，您可以在此看到任務圖、僕從卡片及講述者饋送。
5. **完成** — 建置完成後，會顯示「Nugget 建置完成！」對話框，摘要成果與任何學習時刻。

> **試試看**：從範例選擇器選取「簡易網頁 App」範例。它會在畫布上載入一個目標積木、一個功能積木及一個部署網頁積木。按下綠色的 GO 按鈕，選擇一個資料夾，然後在任務控制中心觀察僕從們如何規劃任務、撰寫程式碼並部署您的 App。

## 後續步驟

- 了解 [工作區編輯器 (Workspace Editor)](Workspace-Editor) 的佈局與控制
- 探索所有可用的 [積木 (Block-Reference)](Block-Reference)
- 了解 [建置生命週期 (Building-a-Nugget)](Building-a-Nugget)
