# Elisa 前端

React + Vite + Blockly SPA。視覺化編輯器與建置監控介面。

## 技術棧

- React 19.2, TypeScript 5.9, Vite 7.3
- Tailwind CSS 4
- Blockly 12.3 (積木編輯器)
- @xyflow/react 12.10 (任務 DAG 視覺化)

## 開發指令

```bash
npm run dev          # 啟動開發伺服器 (連接埠 5173，將 /api 與 /ws 代理至 localhost:8000)
npm run build        # 類型檢查 + 生產環境建置
npm run lint         # ESLint
npm run test         # Vitest (執行一次)
npm run test:watch   # Vitest (監看模式)
```

## 結構

```
src/
  components/
    BlockCanvas/         Blockly 工作區
      BlockCanvas.tsx      編輯器封裝 (建置期間為唯讀)
      blockDefinitions.ts  所有 25 種積木類型定義
      blockInterpreter.ts  工作區 -> ProjectSpec JSON 轉換器
      toolbox.ts           調色盤分組與積木排序
    MissionControl/      右側邊欄 (w-80)
      MissionControl.tsx   包含分頁/面板的容器
      TaskDAG.tsx          @xyflow/react 相依關係圖
      CommsFeed.tsx        滾動式的代理訊息日誌
      MetricsPanel.tsx     各代理的 Token 使用條
    BottomBar/           底部面板 (h-32, 4 個分頁)
      BottomBar.tsx        分頁容器
      GitTimeline.tsx      包含檔案差異的提交列表
      TestResults.tsx      通過/失敗列表 + 覆蓋率條
      BoardOutput.tsx      ESP32 序列埠輸出串流
      TeachingSidebar.tsx  教學時刻清單
    shared/              可重複使用的 UI 元件
      GoButton.tsx         懸浮的建置觸發按鈕
      HumanGateModal.tsx   批准/拒絕彈窗
      QuestionModal.tsx    多選式代理提問彈窗
      TeachingToast.tsx    懸浮通知
      AgentAvatar.tsx      狀態點 + 角色圖示
    Skills/
      SkillsModal.tsx      自定義技能的 CRUD 編輯器 + 範本庫
    Rules/
      RulesModal.tsx       規則的 CRUD 編輯器 + 範本庫
  hooks/
    useBuildSession.ts   WebSocket 連線 + 工作階段狀態
  App.tsx                根佈局，所有最頂層狀態 (useState)
```

## 狀態管理

未使用狀態管理函式庫。所有狀態透過 `useState` 存在於 `App.tsx` 中，並作為 props 向下傳遞。`useBuildSession` hook 管理 WebSocket 連線並將傳入事件分發至狀態設定程序 (setters)。

UI 狀態機：`design` (設計) -> `building` (建置) -> `review` (評閱) -> `deploy` (部署) -> `done` (完成)

## 新增積木類型

1. 遵循現有模式（顏色、欄位、連接）在 `BlockCanvas/blockDefinitions.ts` 中定義積木。
2. 將其新增至 `BlockCanvas/toolbox.ts` 中的適當類別。
3. 在 `BlockCanvas/blockInterpreter.ts` 中新增解釋邏輯，將其映射至 `ProjectSpec`。
4. 重新建置並測試 —— 積木會自動出現在調色盤中。
