# Elisa 後端

Express 5 + WebSocket 伺服器。協調 AI 代理、管理構建工作階段，並處理硬體整合。

## 技術棧

- Express 5, TypeScript 5.9, tsx (執行環境)
- ws 8 (WebSocket)
- simple-git 3 (git 操作)
- serialport 12 (ESP32 通訊)
- @anthropic-ai/sdk (用於規劃器/教學的 Claude API)
- zod 4 (驗證)

## 開發指令

```bash
npm run dev          # 使用 tsx watch 啟動 (連接埠 8000，自動重新載入)
npm run start        # 生產環境啟動
npm run test         # Vitest (執行一次)
npm run test:watch   # Vitest (監看模式)
```

## 結構

```
src/
  server.ts            Express 應用程式、路由註冊、WebSocket 升級
  routes/
    sessions.ts        /api/sessions/* 端點
    hardware.ts        /api/hardware/* 端點
  services/
    orchestrator.ts    中央建置管線控制器
    metaPlanner.ts     ProjectSpec -> 任務 DAG 分解 (Claude API)
    agentRunner.ts     各任務透過 SDK query() API 執行代理
    gitService.ts      各工作階段的 git 儲存庫初始化與提交
    testRunner.ts      pytest 執行與覆蓋率解析
    hardwareService.ts ESP32 偵測/編譯/燒錄/序列埠監控
    teachingEngine.ts  概念課程、去重、Claude Sonnet 備案
```

## 服務架構

**建置管線** (由 Orchestrator 管理)：

1. `MetaPlanner.plan(spec)` -- 呼叫 Claude (模型：`claude-opus-4-6`) 將 ProjectSpec 分解為具備相依關係的任務 DAG。驗證循環依賴。解析失敗時重試。
2. **任務執行迴圈** -- 對於每個就緒任務：
   - `AgentRunner.execute(prompt)` -- 呼叫 SDK `query()` 執行代理。透過非同步迭代串流輸出/Token。超時：300 秒，重試：2 次，模型：`claude-opus-4-6`。
   - `GitService.commit()` -- 提交變更並標記代理。
   - `TeachingEngine.check()` -- 呈現教學時刻 (每個工作階段中每個概念已去重)。備案使用 Claude Sonnet。
3. `TestRunner.runTests()` -- 執行 `pytest tests/ -v --cov=src`。解析輸出。超時：120 秒。
4. `HardwareService.flash()` -- 若目標為 ESP32：偵測 USB、使用 `py_compile` 編譯、透過 `mpremote` 燒錄。超時：60 秒。

所有狀態均存在記憶體中。無資料庫。各工作階段擁有專屬的暫存工作區目錄。

## 新增 API 端點

1. 在 `routes/` 下的適當檔案中新增路由處理程序。
2. 在 `server.ts` 中註冊。
3. 若會發出 WebSocket 事件，請將事件類型新增至類型檔案中的 `WSEvent` 聯集。
4. 更新 [API 參考](../docs/api-reference.md)。

## 新增代理角色

1. 將角色新增至 `AgentRole` 類型。
2. 更新 `MetaPlanner` 提示詞以理解新角色。
3. 在 `AgentRunner` 中配置各角色特有的行為。
4. 預計讓使用者選擇時，在前端新增對應的積木類型。
