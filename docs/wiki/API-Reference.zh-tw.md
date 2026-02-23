# API 參考 (API Reference)

## REST 端點

基礎 URL：`http://localhost:8000/api`

所有端點（除了 `/api/health` 之外）都需要 `Authorization: Bearer <token>` 請求標頭。該權杖 (token) 在伺服器啟動時產生，並透過 IPC 分送給 Electron 渲染進程。

### 工作階段 (Sessions)

| 方法 | 路徑 | 請求主體 | 回應 | 說明 |
|--------|------|--------------|----------|-------------|
| POST | `/sessions` | — | `{ session_id }` | 建立新的建置工作階段 |
| GET | `/sessions/:id` | — | `BuildSession` | 獲取完整的工作階段狀態 |
| POST | `/sessions/:id/start` | `{ spec: NuggetSpec, workspace_path?, workspace_json? }` | `{ status: "started" }` | 帶著 NuggetSpec 開始建置 |
| POST | `/sessions/:id/stop` | — | `{ status: "stopped" }` | 取消正在進行的建置 |
| GET | `/sessions/:id/tasks` | — | `Task[]` | 列出工作階段中的所有任務 |
| GET | `/sessions/:id/git` | — | `CommitInfo[]` | 獲取提交記錄 |
| GET | `/sessions/:id/tests` | — | 測試結果物件 | 獲取測試結果 |
| GET | `/sessions/:id/export` | — | `application/zip` | 將 nugget 目錄匯出為 zip |
| POST | `/sessions/:id/gate` | `{ approved?, feedback? }` | `{ status: "ok" }` | 回應人類閘門 |
| POST | `/sessions/:id/question` | `{ task_id, answers }` | `{ status: "ok" }` | 回答代理提問 |

### 技能 (Skills)

| 方法 | 路徑 | 請求主體 | 回應 | 說明 |
|--------|------|--------------|----------|-------------|
| POST | `/skills/run` | `{ plan: SkillPlan, allSkills? }` | `{ session_id }` | 開始獨立的技能執行 |
| POST | `/skills/:sessionId/answer` | `{ step_id, answers }` | `{ status: "ok" }` | 回答技能的 `ask_user` 問題 |

### 硬體 (Hardware)

| 方法 | 路徑 | 請求主體 | 回應 | 說明 |
|--------|------|--------------|----------|-------------|
| GET | `/hardware/detect` | — | `{ detected, port?, board_type? }` | 偵測已連接的 ESP32 |
| POST | `/hardware/flash/:id` | — | `{ success, message }` | 將工作階段產出燒錄至開發板 |

### 工作區 (Workspace)

| 方法 | 路徑 | 請求主體 | 回應 | 說明 |
|--------|------|--------------|----------|-------------|
| POST | `/workspace/save` | `{ workspace_path, workspace_json?, skills?, rules?, portals? }` | `{ status: "saved" }` | 將設計檔案儲存至目錄 |
| POST | `/workspace/load` | `{ workspace_path }` | `{ workspace, skills, rules, portals }` | 從目錄載入設計檔案 |

### 健康狀態 (Health)

| 方法 | 路徑 | 回應 | 說明 |
|--------|------|----------|-------------|
| GET | `/health` | `{ status, apiKey, apiKeyError?, agentSdk }` | 健康檢查 (不需授權) |

狀態值：`"ready"` (就緒) 或 `"degraded"` (降級)。API 金鑰值：`"valid"`, `"invalid"`, `"missing"`, `"unchecked"`。代理 SDK：`"available"` 或 `"not_found"`。

---

## WebSocket 事件

連線至：`ws://localhost:8000/ws/session/:sessionId?token=<auth_token>`

所有事件皆以包含 `type` 辨識欄位的 JSON 格式，從伺服器流向用戶端。

### 工作階段生命週期

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `planning_started` | — | 元規劃器正在分解規格 |
| `plan_ready` | `{ tasks, agents, explanation, deployment_target? }` | 任務 DAG 已就緒 |
| `workspace_created` | `{ nugget_dir }` | 已建立 Nugget 工作區目錄 |
| `session_complete` | `{ summary }` | 建置已完成 |
| `error` | `{ message, recoverable }` | 發生錯誤 |

### 任務執行

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `task_started` | `{ task_id, agent_name }` | 代理開始執行任務 |
| `task_completed` | `{ task_id, summary }` | 任務成功完成 |
| `task_failed` | `{ task_id, error, retry_count }` | 任務失敗 (可能會自動重試) |

### 代理活動

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `agent_output` | `{ task_id, agent_name, content }` | 串流中的代理訊息片段 |
| `agent_status` | `{ agent }` | 代理狀態已變更 |
| `agent_message` | `{ from, to, content }` | 代理間通訊 |
| `token_usage` | `{ agent_name, input_tokens, output_tokens, cost_usd }` | Token 消耗量 |
| `budget_warning` | `{ total_tokens, max_budget, cost_usd }` | 達到預算門檻 |
| `minion_state_change` | `{ agent_name, old_status, new_status }` | 僕從狀態轉換 |
| `narrator_message` | `{ from, text, mood, related_task_id? }` | 講述者評論 |
| `permission_auto_resolved` | `{ task_id, permission_type, decision, reason }` | 權限已自動解析 |

### 建置產物

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `commit_created` | `{ sha, message, agent_name, task_id, timestamp, files_changed }` | Git 提交已建立 |
| `test_result` | `{ test_name, passed, details }` | 個別測試結果 |
| `coverage_update` | `{ percentage, details? }` | 程式碼覆蓋率報告 |

### 技能執行

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `skill_started` | `{ skill_id, skill_name }` | 技能執行已開始 |
| `skill_step` | `{ skill_id, step_id, step_type, status }` | 技能步驟進度 |
| `skill_question` | `{ skill_id, step_id, questions }` | 技能詢問使用者問題 |
| `skill_output` | `{ skill_id, step_id, content }` | 技能步驟輸出 |
| `skill_completed` | `{ skill_id, result }` | 技能已完成 |
| `skill_error` | `{ skill_id, message }` | 技能失敗 |

### 部署

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `deploy_started` | `{ target }` | 部署階段開始 |
| `deploy_progress` | `{ step, progress }` | 部署進度 (0–100) |
| `deploy_checklist` | `{ rules }` | 部署前規則檢查清單 |
| `deploy_complete` | `{ target, url? }` | 部署已完成 |
| `serial_data` | `{ line, timestamp }` | ESP32 序列埠監控輸出 |

### 使用者互動

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `human_gate` | `{ task_id, question, context }` | 建置暫停，等待批准 |
| `user_question` | `{ task_id, questions }` | 代理詢問使用者問題 |
| `teaching_moment` | `{ concept, headline, explanation, tell_me_more?, related_concepts? }` | 學習時刻 |

---

## NuggetSpec 結構 (Schema)

由積木解釋器產出並傳送至 `POST /sessions/:id/start` 的 JSON 結構。

```typescript
interface NuggetSpec {
  project: {
    goal: string;           // 使用者想要構建什麼
    description: string;    // 詳細說明
    type: string;           // "game" | "website" | "hardware" | "story" | "tool" | "general"
  };
  requirements: Array<{
    type: string;           // "feature" | "constraint" | "when_then" | "data"
    description: string;
  }>;
  style?: {
    visual: string | null;
    personality: string | null;
  };
  agents: Array<{
    name: string;
    role: string;           // "builder" | "tester" | "reviewer" | "custom"
    persona: string;
  }>;
  hardware?: {
    target: string;
    components: Array<{ type: string; [key: string]: unknown }>;
  };
  deployment: {
    target: string;         // "preview" | "web" | "esp32" | "both"
    auto_flash: boolean;
  };
  workflow: {
    review_enabled: boolean;
    testing_enabled: boolean;
    human_gates: string[];
    flow_hints?: Array<{ type: "sequential" | "parallel"; descriptions: string[] }>;
    iteration_conditions?: string[];
  };
  skills?: Array<{ id: string; name: string; prompt: string; category: string }>;
  rules?: Array<{ id: string; name: string; prompt: string; trigger: string }>;
}
```

### 關鍵類型

```typescript
type TaskStatus = "pending" | "in_progress" | "done" | "failed";
type AgentRole = "builder" | "tester" | "reviewer" | "custom";
type AgentStatus = "idle" | "working" | "done" | "error" | "waiting";
type SessionState = "idle" | "planning" | "executing" | "testing" | "deploying" | "reviewing" | "done";
```

---

## SkillPlan 結構 (Schema)

傳送至 `POST /api/skills/run`。代表要執行的步驟序列。

```typescript
interface SkillPlan {
  skillId: string;
  skillName: string;
  steps: SkillStep[];       // 最多 50 個步驟
}
```

### SkillStep 類型

| 類型 | 欄位 | 說明 |
|------|--------|-------------|
| `ask_user` | `question`, `header`, `options`, `storeAs` | 暫停執行，呈現選項給使用者 |
| `branch` | `contextKey`, `matchValue`, `thenSteps` | 根據上下文的值進行條件式執行 |
| `invoke_skill` | `skillId`, `storeAs` | 呼叫另一個技能 (最大深度 10，循環偵測) |
| `run_agent` | `prompt`, `storeAs` | 帶著提示詞範本產生一個 Claude 代理 |
| `set_context` | `key`, `value` | 設定上下文變數 |
| `output` | `template` | 產生最終的技能輸出 |

所有文字欄位均支援 `{{key}}` 範本語法，用以揷入上下文變數。

---

## 講述者系統 (Narrator System)

講述者透過 Claude Haiku 將原始建置事件轉化為適合兒童的評論。

- **觸發事件**：`task_started`, `task_completed`, `task_failed`, `agent_message`, `error`, `session_complete`
- **情緒 (Moods)**：`excited`, `encouraging`, `concerned`, `celebrating`
- **頻率限制**：每個任務每 15 秒最多傳送 1 條訊息
- **去抖動 (Debounce)**：按任務累積 `agent_output` 事件，並在 10 秒靜默期後進行轉譯
- **模型**：可透過 `NARRATOR_MODEL` 環境變數配置 (預設：`claude-haiku-4-5-20241022`)

---

## 門戶安全 (Portal Security)

### 指令允許清單

CLI 門戶會根據以下清單驗證指令：`node`, `npx`, `python`, `python3`, `pip`, `pip3`, `mpremote`, `arduino-cli`, `esptool`, `git`

### 執行模型

`CliPortalAdapter.execute()` 使用 `execFile` (不必經過 Shell) — 參數會直接傳遞給可執行檔案，防止 Shell 注入。

### 序列埠驗證

序列埠門戶在燒錄操作前，會透過 USB VID:PID 開防板偵測進行驗證。
