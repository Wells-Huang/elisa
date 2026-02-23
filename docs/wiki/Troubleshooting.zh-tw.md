# 疑難排解 (Troubleshooting)

## 就緒勳章 (Readiness Badge)

標題列右上角的就緒勳章會告訴您 Elisa 是否已準備好開始建置：

| 勳章 | 顏色 | 代表意義 | 修復方法 |
|-------|-------|---------|-----|
| **就緒 (Ready)** | 綠色 | 所有項目皆已配置且運作正常 | 無需動作 |
| **檢查中 (Checking...)** | 灰色 | Elisa 正在測試連線 | 請稍候片刻 |
| **需要 API 金鑰** | 黃色 | 未設定 API 金鑰，或金鑰無效 | 請參閱下方的「API 金鑰問題」 |
| **尚未就緒 (Not Ready)** | 黃色 | 後端正在執行但發生問題 | 查看工具提示以獲取詳情 |
| **離線 (Offline)** | 紅色 | 無法連線至後端伺服器 | 請參閱下方的「後端未執行」 |

## API 金鑰問題

**「找不到 API 金鑰」**
- 在 Electron 應用程式中：前往設定選單並輸入您的 Anthropic API 金鑰。
- 在開發模式下：啟動後端前，先設定 `ANTHROPIC_API_KEY` 環境變數。

**「API 金鑰無效」**
- 檢查是否複製了完整的金鑰（以 `sk-ant-` 開頭）。
- 到 [console.anthropic.com](https://console.anthropic.com) 檢查金鑰是否仍有效。
- 確保文字中沒有多餘的空格或換行符號。

## 後端未執行

若勳章顯示「離線 (Offline)」：
- **Electron 應用程式**：後端會自動啟動。請嘗試重啟應用程式。
- **開發模式**：確保後端正在執行，預設連接埠為 8000。
- 檢查終端機視窗中的錯誤訊息。

## WebSocket 連線中斷

前端每 3 秒會自動嘗試重新連線。若您發現狀態更新遺失：
- 檢查後端是否仍在執行。
- 若後端發生當機，請重啟後端並開始新的建置（舊的工作階段資料將會遺失）。

## 建置卡住或發生錯誤

**建置似乎卡住了**
- 檢查底部欄的「進度」標籤以確認目前的階段。
- 檢查「Token」標籤 — 若使用量接近預算上限（預設為 50 萬 Token），建置可能已被停止。
- 按下 STOP 並嘗試簡化您的設計後再次建置。

**出現錯誤橫幅**
- 頂端會出現紅色橫幅顯示錯誤訊息。點擊 X 可將其關閉。
- 「Elisa 無法連線至她的 AI 大腦」— 代表 API 金鑰缺失或無效。
- 建置錯誤會包含失敗原因的詳細資訊。
- 對於驗證錯誤，訊息會標註出哪個欄位有問題。

## 偵測不到開發板

- 檢查 USB 線（部分線材僅供充電，無資料傳輸功能）。
- 換一個 USB 插槽試試。
- 安裝開發板晶片的 USB 驅動程序（CP210x, CH9102）。
- 在 Windows 上，開啟「裝置管理員」並在「連接埠 (Ports)」下方尋找 COM 連接埠。
- 詳見 [硬體整合 (Hardware Integration)](Hardware-Integration.zh-tw.md) 獲取完整的支援開發板清單。

## 連接埠衝突

- 後端預設使用連接埠 8000。若已被佔用，請更改環境變數中的 `PORT`。
- 開發模式下，前端預設使用連接埠 5173。
- 兩個連接埠都必須可用，Elisa 才能在開發模式下運作。

## 測試失敗並顯示 "pytest not found"

請安裝 Python 與 pytest：

```bash
pip install pytest pytest-cov
```

測試執行器會呼叫 `pytest tests/ -v --cov=src`。`python` 與 `pytest` 都必須在您的系統 PATH 路徑中。

## Token 預算超出上限

每個建置工作階段預設有 500,000 Token 的預算。當使用量達到 80% 時，會出現警告；當超出預算時，建置會優雅停止。

要在預算內完成工作：
- 簡化設計，減少不必要的功能。
- 使用建置完成後的「繼續工作」選項進行迭代，而非重頭開始。
- 減少代理僕從的數量 — 每位代理都會獨立消耗 Token。

## 常見錯誤訊息

| 錯誤訊息 | 代表意義 | 修復方法 |
|-------|---------|-----|
| "Elisa can't connect to her AI brain" | API 金鑰問題 | 檢查您的 Anthropic API 金鑰 |
| "Elisa couldn't get ready to build" | 工作階段建立失敗 | 重新啟動後端 |
| "No ESP32 board detected" | 硬體燒錄失敗 | 連接開發板，檢查 USB 線 |
| "mpremote not found" | 遺失燒錄工具 | 執行 `pip install mpremote` |
| "Flash timed out after 60 seconds" | 開發板無回應 | 重啟開發板並再試一次 |
| "Agent SDK not installed" | 缺失相依套件 | 在後端資料夾執行 `npm install` |
