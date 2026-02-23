# 入門指南

## 前提條件

| 需求 | 版本 | 備註 |
|-------------|---------|-------|
| Node.js | 20+ | 必要 |
| npm | 10+ | 隨 Node.js 安裝 |
| `ANTHROPIC_API_KEY` | -- | 環境變數。請至 [console.anthropic.com](https://console.anthropic.com) 獲取 |
| Claude Agent SDK | -- | 在後端執行 `npm install` 時會自動安裝。 |
| Python + pytest | 3.10+ | 選填。僅當建置包含測試任務時需要。 |
| ESP32 + mpremote | -- | 選填。僅用於硬體部署。執行 `pip install mpremote` |

## 安裝與執行

```bash
git clone https://github.com/zoidbergclawd/elisa.git
cd elisa
npm install          # 自動安裝根目錄、後端及前端的相依套件
npm run dev          # 啟動後端 (連接埠 8000) + 前端 (連接埠 5173)
```

在瀏覽器中開啟 `http://localhost:5173`。

若要以桌面應用程式運行：`npm run dev:electron`

## 首次建置工作階段

1. **設計 (Design)** -- 從調色盤中將積木拖曳到畫布上。至少需新增一個 `project_goal` 積木，並描述您想要構建的專案。
2. **點擊 Go** -- 大型的懸浮按鈕會將您的積木工作區以 ProjectSpec JSON 格式發送至後端。
3. **觀察 (Watch)** -- 右側側欄會顯示任務相依性圖示。代理會出現、認領任務並串流輸出。底部欄會顯示 git 提交紀錄、測試結果及教學時刻。
4. **互動 (Interact)** -- 若您放置了「與我確認」積木，建置會暫停並顯示彈窗詢問您的批准。代理也可能在建置中途提出問題。
5. **完成 (Done)** -- 工作階段完成後，您將擁有一個具備 git 歷程的可用專案。

## 技能與規則

首次建置後，嘗試透過自訂技能與規則來擴展您的代理。

**建立技能** -- 從側欄（扳手圖示）開啟技能彈窗。給予名稱、提示詞及類別（`agent`, `feature` 或 `style`）。在畫布上拖入「使用技能」積木，即可將其包含在下次建置中。

**建立規則** -- 從側欄（盾牌圖示）開啟規則彈窗。規則是自動觸發的護欄（`always`, `on_task_complete`, `on_test_fail`, `before_deploy`）。在畫布上拖入「套用規則」積木。

**複合技能** -- 在技能彈窗內開啟流程編輯器，將各步驟鏈接起來：詢問使用者問題、根據答案分支、執行代理，以及調用其他技能。使用 `{{key}}` 語法在各步驟間引用上下文變數。

技能與規則在側欄擁有獨立的按鈕以便快速存取。

## 疑難排解

**後端無法啟動** -- 確認環境變數中已設定 `ANTHROPIC_API_KEY`。後端需要它來執行元規劃器與教學引擎。

**WebSocket 連線中斷** -- 前端會自動重新連線。若後端當機，請重啟後端並建立新的工作階段。

**偵測不到 ESP32** -- 檢查 USB 連線。支援的晶片包含：CP210x (Heltec WiFi LoRa 32 V3), ESP32-S3 原生 USB, CH9102。

**測試失敗並顯示 "pytest not found"** -- 請安裝 Python 與 pytest：`pip install pytest pytest-cov`。測試執行器會呼叫 `pytest tests/ -v --cov=src`。

**連接埠衝突** -- 後端預設為 8000，前端預設為 5173。可分別於 `backend/src/server.ts` 及 `frontend/vite.config.ts` 中修改。
