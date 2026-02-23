# 開發指南 (Development Guide)

## 前提條件

- Node.js 20+
- npm 10+
- Python 3.10+（選填，供測試執行器使用）
- `ANTHROPIC_API_KEY` 環境變數

## 設定

```bash
git clone https://github.com/zoidbergclawd/elisa.git
cd elisa
npm install              # 安裝根目錄 + 後端 + 前端相依套件
npm run dev:electron     # 啟動後端、前端與 Electron 視窗
```

## npm 指令碼

### 根目錄 (儲存庫根部)

| 指令碼 | 作用 |
|--------|-------------|
| `npm run dev` | 啟動後端 + 前端（僅限瀏覽器，無 Electron） |
| `npm run dev:electron` | 啟動後端 + 前端 + Electron 視窗 |
| `npm run build:frontend` | Vite 生產環境建置 → `frontend/dist/` |
| `npm run build:backend` | esbuild 打包 → `backend/dist/server-entry.js` |
| `npm run build:electron` | tsc 編譯 `electron/main.ts` + `preload.ts` |
| `npm run dist` | electron-builder → 產出安裝程式 (NSIS/DMG) |

### 後端 (`cd backend`)

| 指令碼 | 作用 |
|--------|-------------|
| `npm run dev` | 使用 tsx 監聽模式啟動（連接埠 8000，自動重新載入） |
| `npm run start` | 生產環境啟動 |
| `npm run test` | Vitest（單次執行） |
| `npm run test:watch` | Vitest（監聽模式） |

### 前端 (`cd frontend`)

| 指令碼 | 作用 |
|--------|-------------|
| `npm run dev` | 開發伺服器（連接埠 5173，將 /api 與 /ws 代理至後端） |
| `npm run build` | 類型檢查 + 生產環境建置 |
| `npm run lint` | ESLint 檢查 |
| `npm run test` | Vitest（單次執行） |
| `npm run test:watch` | Vitest（監聽模式） |

## 環境變數

| 變數名稱 | 必要 | 預設值 | 用途 |
|----------|----------|---------|---------|
| `ANTHROPIC_API_KEY` | 是 | — | Claude API/SDK 存取金鑰 |
| `CLAUDE_MODEL` | 否 | `claude-opus-4-6` | 覆蓋代理使用的模型 |
| `NARRATOR_MODEL` | 否 | `claude-haiku-4-5-20241022` | 覆蓋講述者使用的模型 |
| `CORS_ORIGIN` | 否 | `http://localhost:5173` | 在開發環境覆蓋 CORS 來源 |
| `PORT` | 否 | `8000` | 後端連接埠 |

## 專案結構

```
elisa/
  package.json          根目錄：Electron 相依性、建置/開發指令碼
  electron/             Electron 主進程、preload、設定
  frontend/             React 19 + Vite 7 SPA
    src/components/     UI 元件樹
    src/hooks/          React hooks (session, health, WebSocket)
    src/lib/            工具類 (nugget 檔案, 技能範本)
    src/types/          TypeScript 定義
  backend/              Express 5 + TypeScript 伺服器
    src/routes/         REST 端點處理程序
    src/services/       核心服務 (協調器, 執行器, 硬體)
    src/services/phases/ 管線階段 (規劃, 執行, 測試, 部署)
    src/prompts/        代理角色提示詞 + 課程內容
    src/utils/          DAG, 驗證, 日誌, Token 相關工具
    src/tests/          後端測試
  hardware/             ESP32 範本 + 共享的 MicroPython 函式庫
  scripts/              建置工具 (esbuild 打包器)
  docs/                 說明文件
```

## 測試

後端與前端皆使用 Vitest。

- **後端測試**：`cd backend && npm test`
- **前端測試**：`cd frontend && npm test`
- 測試檔案與原始碼位於同一目錄下（`.test.ts` / `.test.tsx`）
- 行為/整合測試位於 `backend/src/tests/behavioral/`

### 臭蟲修復測試要求

每個臭蟲 (bug) 的修復 **必須** 包含測試案例，用以：
1. 重現失敗（若臭蟲再次出現，該測試應會失敗）
2. 證明修復有效

## 如何新增 API 端點

1. 在 `backend/src/routes/` 下的適當檔案中新增路由處理程序。
2. 在 `backend/src/server.ts` 中註冊該路由。
3. 若該端點會發出 WebSocket 事件，請將事件類型新增至 types 檔案中的 `WSEvent` 聯集類型中。
4. 更新 [API 參考 (API Reference)](API-Reference)。
5. 更新 `backend/CLAUDE.md` 中的 API 表格。

## 如何新增積木類型

1. 在 `frontend/src/components/BlockCanvas/blockDefinitions.ts` 中依循現有模式（顏色、欄位、連接方式）定義積木。
2. 在 `frontend/src/components/BlockCanvas/toolbox.ts` 中將其加入適當類別。
3. 在 `frontend/src/components/BlockCanvas/blockInterpreter.ts` 中新增解釋邏輯，將其映射至 NuggetSpec。
4. 重新建置並測試 —— 積木會自動出現在調色盤中。

## 如何新增代理角色

1. 將該角色新增至後端 types 中的 `AgentRole` 類型。
2. 在 `backend/src/prompts/` 中建立新的提示詞範本。
3. 更新 `MetaPlanner` 提示詞以使其理解新角色。
4. 在 `AgentRunner` 中配置任何角色特定的行為。
5. 若使用者應能選取該角色，請在前端新增對應的積木類型。

## 慣例

- 無資料庫。工作階段狀態存於記憶體，提供選用的 JSON 持久化。
- 每個代理任務皆透過 Claude Agent SDK 的 `query()` API 運行。
- 前端透過 REST (指令) + WebSocket (事件) 通訊。
- Blockly 工作區 → NuggetSpec JSON (經 Zod 驗證) → 後端協調管線。
- 當 DAG 相依關係允許時，最多可同時執行 3 個任務。
- 僅使用 Functional components，所有樣式皆使用 Tailwind 輔助類別。
- 狀態顏色：藍色=工作中，綠色=完成，紅色=錯誤，黃色=警告。
