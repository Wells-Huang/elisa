# 前端模組

React 19 + TypeScript + Vite SPA。用於組合 nugget 規格的視覺化積木編輯器 (Blockly)，以及用於監控代理執行的即時儀表板。

## 技術棧

- React 19.2, Vite 7.3, TypeScript 5.9, Tailwind CSS 4
- Blockly 12.3 (積木編輯器), @xyflow/react 12.10 (任務 DAG 視覺化), elkjs (圖表佈局)
- JSZip 3 (nugget 檔案儲存/讀取)
- Vitest + Testing Library (測試)

## 結構

```
src/
  App.tsx                    根元件。擁有所有工作階段狀態。分頁式主佈局。
  main.tsx                   入口點。React 19 createRoot。
  components/
    BlockCanvas/             Blockly 編輯器 + 積木轉 NuggetSpec 轉換器 + 工作區側欄
    AgentTeam/               全寬代理卡片 + 通訊饋送面板 (代理分頁)
    TaskMap/                 全寬互動式任務 DAG 面板 (任務分頁)
    MissionControl/          MissionControlPanel (佈局)、MinionSquadPanel、NarratorFeed、TaskDAG、CommsFeed、指標面板
    BottomBar/               底部標籤：時間軸、測試、開發板、學習、進度、Token
    Skills/                  技能編輯器彈窗 + 範本庫 + SkillFlowEditor (視覺化流程編輯器)
    Rules/                   規則編輯器彈窗 + 範本庫
    Portals/                 門戶編輯器彈窗 + 註冊表
    shared/                  主標籤欄、執行按鈕、人類閘門彈窗、提問彈窗、教學提示、代理頭像、就緒勳章、範例選擇彈窗、目錄選擇彈窗
  hooks/
    useBuildSession.ts       所有建置工作階段狀態 (任務、代理、提交、事件等)
    useSkillSession.ts       獨立技能執行狀態 + WebSocket 事件
    useBoardDetect.ts        透過 /api/hardware/detect 進行 ESP32 開發板偵測輪詢
    useHealthCheck.ts        輪詢 /api/health 以檢查後端就緒狀態 (API 金鑰 + SDK 狀態)
    useWebSocket.ts          具備自動重新連線功能的 WebSocket 連線 (3 秒間隔)
  lib/
    nuggetFile.ts            .elisa nugget 檔案儲存/讀取工具 (基於 JSZip)
    playChime.ts             用於開發板偵測事件的 Web Audio API 雙音提示音
    skillTemplates.ts        用於範本庫的內建技能與規則範本
    terminology.ts           適合兒童的術語映射 (技術術語 -> 親切的標籤)
    examples/                內建範例 nuggets (ES 模組，可離線使用)
  types/
    index.ts                 所有 TypeScript 介面 (NuggetSpec, Task, Agent, WSEvent 等)
```

## 狀態管理

未使用狀態管理函式庫。`useBuildSession` hook 以 `useState` 變數持有所有工作階段狀態。WebSocket 事件抵達後透過 `handleEvent()` 分發並更新相關狀態片斷。

工作區 JSON、技能與規則在每次變更時自動儲存至 `localStorage`，並在頁面載入時恢復。金鑰：`elisa:workspace`, `elisa:skills`, `elisa:rules`, `elisa:portals`, `elisa:workspace-path` (使用者選擇的目錄)。

UI 階段：`design` (設計) | `building` (建置) | `review` (評閱) | `deploy` (部署) | `done` (完成)

主要分頁：`workspace` (工作區) | `agents` (代理) | `tasks` (任務) (建置開始時自動切換至 `agents`)

## 與後端通訊

- **REST**：`POST /api/sessions`, `POST /api/sessions/:id/start` (可接受選用的 `workspace_path`), `POST /api/sessions/:id/stop` (取消建置), `POST /api/sessions/:id/gate`, `POST /api/sessions/:id/question`, `GET /api/sessions/:id/export`, `POST /api/workspace/save`, `POST /api/workspace/load`
- **WebSocket**：`ws://localhost:8000/ws/session/:sessionId` - 接收所有串流事件
- 在開發模式下，Vite 將 `/api/*` 與 `/ws/*` 代理至後端

## 關鍵規範

- 僅使用功能元件，每個元件擁有專屬 Props 介面
- 所有樣式均使用 Tailwind 工具類別
- 狀態顏色：藍色=進行中，綠色=完成，紅色=錯誤，黃色=警告
- 測試檔案與原始碼位於同一目錄 (`.test.tsx` / `.test.ts`)
- WSEvent 為可辨識聯集 (Discriminated Union) - 事件處理程序中使用詳盡的 switch 判斷
- 序列埠列數限制為 MAX_SERIAL_LINES=1000 (超過時移除最舊記錄)
- 後端 Zod 驗證錯誤透過 `body.detail` + `body.errors` 陣列呈現
