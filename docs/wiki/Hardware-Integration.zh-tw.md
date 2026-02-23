# 硬體整合 (Hardware Integration)

Elisa 可以編譯 MicroPython 程式碼，並透過 USB 燒錄至 ESP32 開發板。

## 您需要的裝備

- 一塊 ESP32 開發板（請參閱下方支援清單）
- 一條將開發板連接至電腦的 USB 線（必須支援資料傳輸，不支援僅供充電的線材）
- 適用於開發板的 USB 驅動程序（多數會自動安裝）
- 安裝了 `mpremote` 的 Python 3.10+：`pip install mpremote`

## 支援的開發板

Elisa 透過 USB 供應商/產品 ID (VID:PID) 來識別以下開發板：

| 開發板 | USB 晶片 | VID:PID |
|-------|----------|---------|
| Heltec WiFi LoRa 32 V3 | CP210x | 10C4:EA60 |
| ESP32-S3 (原生 USB) | 內建 CDC | 303A:1001 / 303A:4001 |
| ESP32 (CH9102) | CH9102 | 1A86:55D4 |
| 任何 Espressif 裝置 | 原生 USB | 303A:* |

不在清單中的開發板仍可能運作 —— Elisa 會嘗試探測序列埠以尋找 MicroPython REPL 提示符。

## 開發板自動偵測

當您處於設計模式時，Elisa 會定期掃描 USB 序列埠以檢查已連接的開發板。偵測使用 VID:PID 比對（速度快，不會佔用連接埠）。

當插入新開發板時：

1. 播放提示音。
2. 出現「開發板已連接！」彈窗，顯示開發板類型與連接埠。
3. 您可以點擊 **建立門戶 (Create Portal)** 來自動為該開發板設定序列埠 [門戶 (Portals)](Portals)。
4. 或點擊 **稍後再說 (Maybe later)** 關閉彈窗（同一塊開發板在重新啟動前不會再次觸發彈窗）。

若該連接埠已存在門戶，彈窗則會顯示「查看門戶 (View Portals)」。

## 在設計中使用硬體

要構建硬體專案：

1. 新增 **目標 (Goal)** 積木，描述您希望開發板執行什麼工作。
2. （選填）新增 **範本 (Template)** 積木並設定為「硬體 Nugget」。
3. 新增 **部署到 ESP32 (Deploy ESP32)** 積木（或選用「兩者皆部署」以同時處理網頁與硬體）。
4. （選填）為您的開發板建立序列埠門戶（以便使用「告訴/當...時/詢問」積木）。
5. 按下 **GO**。

## 硬體建置過程中會發生什麼事

1. **規劃 (Planning)** — 代理規劃 MicroPython 任務。
2. **編碼 (Coding)** — 構建代理撰寫 `.py` 檔案。
3. **編譯 (Compile)** — Elisa 對每個 Python 檔案執行 `py_compile` 以檢查語法錯誤。
4. **燒錄 (Flash)** — 檔案被傳送至開發板。Elisa 會先嘗試序列埠貼上模式（適用於所有 MicroPython 開發板，包含 ESP32-S3 原生 USB）；若有需要，則會退而使用 `mpremote`。燒錄超時時間為 60 秒，最多重試 3 次。
5. **序列埠監控** — 燒錄完成後，底部標籤列的「開發板」標籤會顯示運行中程式的序列埠輸出。

## 硬體抽象函式庫

Elisa 提供了一個共享的 MicroPython 函式庫 (`hardware/lib/elisa_hardware.py`)，並會指示代理由此進行開發。它為常見操作提供了簡化的抽象，例如 LED 控制、按鈕輸入、LoRa 通訊及感測器讀取。

## 「開發板」標籤頁

底部欄的「開發板」標籤以 115200 波特率顯示來自已連接開發板的即時序列埠輸出。每行都會顯示時間戳記與輸出文字。顯示上限為 1,000 行（最舊的行會被移除）。

## 疑難排解

**偵測不到開發板**
- 檢查 USB 線（部分線材僅供充電，無資料傳輸功能）。
- 換一個 USB 插槽試試。
- 安裝開發板晶片的 USB 驅動程序（Heltec 使用 CP210x 驅動，部分 ESP32 使用 CH9102 驅動）。
- 在 Windows 上，開啟「裝置管理員」並查看 COM 連接埠。

**燒錄失敗**
- 確保沒有其他程式（序列埠監控、Arduino IDE）佔用了該連接埠。
- 嘗試按下開發板上的 RST (重啟) 按鈕後再次燒錄。
- 檢查是否已安裝 Python 與 mpremote：`pip install mpremote`。

**沒有 MicroPython REPL**
- 您的開發板可能尚未安裝 MicroPython 韌體。請先至 [micropython.org](https://micropython.org/download/) 下載並燒錄韌體。

**序列埠監控沒顯示內容**
- 按下開發板上的 RST 按鈕重啟程式。
- 確保您的 MicroPython 程式碼中包含 `print()` 敘述。
