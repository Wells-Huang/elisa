# 可行性更新：Electron 桌面應用程式與硬體服務

**日期：** 2026-02-11
**分支：** `feature/electron-desktop-app`
**狀態：** 進行中 / 分析中

## 1. 硬體服務分析 (`hardwareService.ts`)

目前實作是外部 CLI 工具的 Node.js 封裝。

*   **依賴鏈：**
    *   Node.js (後端) -> `serialport` (原生模組) -> `mpremote` (Python CLI) -> 硬體。
*   **目前的燒錄策略：**
    *   使用 `mpremote connect <port> cp <files> ...` 來同步 Python 檔案。
    *   使用 `python -m py_compile` 進行語法檢查。
    *   **主要瓶頸：** 需要終端使用者在系統 PATH 中安裝並提供 Python 和 `mpremote`。這違背了自包含「一鍵點擊」桌面應用程式的初衷。

## 2. Electron 的影響與機會

轉向 Electron 是否能簡化 USB/序列埠存取？**是的，但需要更改架構。**

### 「原生模組」陷阱
目前的程式碼使用 `serialport`（原生 Node.js 模組）。在 Electron 中，這需要：
1.  為特定的 Electron 版本重新編譯 `serialport` (`electron-rebuild`)。
2.  處理複雜的打包/簽署問題。

### 機會：Web Serial API
Electron 原生支持 [Web Serial API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Serial_API)。

*   **提案：** 重構 `HardwareService` 以使用 Web Serial（可透過通用繫結在 Electron 渲染進程或主進程中使用），而不是 `serialport` Node 模組。
*   **優點：** 零原生依賴。在 Windows/Mac 更新時不會出現編譯錯誤。

### 機會：內建燒錄功能
不應再調用 `mpremote`（需要 Python），我們應該整合一個純 JavaScript 的 MicroPython 互動函式庫。
*   **待研究的函式庫：** `micropython-ctl` 或改編自 [WebREPL](https://github.com/micropython/webrepl) 專案的程式碼。
*   **可行性：** 高。這使得 Electron 應用程式能夠完全自包含（無外部 Python 需求）。

## 3. 裝置偵測 (`KNOWN_BOARDS`)

目前的清單過於受限：
```typescript
['10C4:EA60', 'Heltec WiFi LoRa 32 V3 (CP210x)'],
['303A:1001', 'ESP32-S3 Native USB'],
['1A86:55D4', 'ESP32 (CH9102)'],
```

**建議：**
1.  **擴充清單：** 加入常見的通用驅動程式 ID（例如 CP2102、CH340），這涵蓋了 90% 的廉價 ESP32 開發板。
2.  **啟發式偵測：** 與其進行精確的 VID:PID 匹配，不如列出所有 COM 連接埠並嘗試握手（發送 `Ctrl-C` 進行中斷，尋找 `>>>` MicroPython 提示字元）以確認該裝置是 MicroPython 開發板。

## 4. MicroPython 整合 (`elisa_hardware.py`)

硬體函式庫很穩固，但依賴於 `sx1262`。
*   **問題：** `sx1262` 驅動程式並非通用 MicroPython 韌體的標準配備。
*   **修正：** `HardwareService` 需要一個「引導 (Bootstrap)」功能。連接到新開發板時，它應在嘗試運行使用者程式碼之前，先自動上傳 `sx1262.py` 驅動檔案。目前它僅上傳工作目錄中的檔案。

## 5. 後續步驟

1.  **POC：** 在 Electron 前端使用 `navigator.serial` (Web Serial) 建立一個小腳本，證明我們可以在不使用 `serialport` 的情況下讀取 MicroPython REPL。
2.  **重構：** 重寫 `flash` 方法，直接透過 Serial (REPL raw paste mode) 寫入檔案，而不是產生 `mpremote` 進程。
3.  **引導：** 將 `sx1262.py` 加入打包資源中，並在第一次燒錄時注入。

**結論：** 繼續使用 Electron，但**放棄** `serialport` 和 `mpremote`，轉而採用純 JS 的 Serial 實作。這將顯著降低使用者的進入門檻。
