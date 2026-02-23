# API 參考

## REST 端點

基礎 URL：`http://localhost:8000/api`

### 工作階段 (Sessions)

| 方法 | 路徑 | 請求主體 | 回應 | 說明 |
|--------|------|--------------|----------|-------------|
| POST | `/sessions` | -- | `{ session_id: string }` | 建立新的建置工作階段 |
| GET | `/sessions/:id` | -- | `BuildSession` | 獲取完整的工作階段狀態 |
| POST | `/sessions/:id/start` | `{ spec: NuggetSpec, workspace_path?: string, workspace_json?: object }` | `{ status: "started" }` | 帶著 NuggetSpec 開始建置 |
| POST | `/sessions/:id/stop` | -- | `{ status: "stopped" }` | 取消正在進行的建置 |
| GET | `/sessions/:id/tasks` | -- | `Task[]` | 列出工作階段中的所有任務 |
| GET | `/sessions/:id/git` | -- | `CommitInfo[]` | 獲取提交記錄 |
| GET | `/sessions/:id/tests` | -- | 測試結果物件 | 獲取測試結果 |
| GET | `/sessions/:id/export` | -- | `application/zip` | 將 nugget 目錄匯出為 zip |
| POST | `/sessions/:id/gate` | `{ approved?: boolean, feedback?: string }` | `{ status: "ok" }` | 回應人類閘門 |
| POST | `/sessions/:id/question` | `{ task_id: string, answers: Record<string, any> }` | `{ status: "ok" }` | 回答代理的提問 |

### 技能 (Skills)

| 方法 | 路徑 | 請求主體 | 回應 | 說明 |
|--------|------|--------------|----------|-------------|
| POST | `/skills/run` | `{ plan: SkillPlan, allSkills?: SkillSpec[] }` | `{ session_id: string }` | 開始獨立的技能執行。`invoke_skill` 步驟需要 `allSkills`。 |
| POST | `/skills/:sessionId/answer` | `{ step_id: string, answers: Record<string, any> }` | `{ status: "ok" }` | 回答技能的 `ask_user` 問題。`answers` 以步驟標題為鍵。 |

### 硬體 (Hardware)

| 方法 | 路徑 | 請求主體 | 回應 | 說明 |
|--------|------|--------------|----------|-------------|
| GET | `/hardware/detect` | -- | `{ detected: boolean, port?: string, board_type?: string }` | 偵測已連接的 ESP32 |
| POST | `/hardware/flash/:id` | -- | `{ success: boolean, message: string }` | 將工作階段產出燒錄至開發板 |

### 工作區 (Workspace)

| 方法 | 路徑 | 請求主體 | 回應 | 說明 |
|--------|------|--------------|----------|-------------|
| POST | `/workspace/save` | `{ workspace_path, workspace_json?, skills?, rules?, portals? }` | `{ status: "saved" }` | 將設計檔案儲存至目錄 |
| POST | `/workspace/load` | `{ workspace_path }` | `{ workspace, skills, rules, portals }` | 從目錄載入設計檔案 |

### 其他

| 方法 | 路徑 | 回應 | 說明 |
|--------|------|----------|-------------|
| GET | `/health` | `{ status: "ready"\|"degraded", apiKey: "valid"\|"invalid"\|"missing"\|"unchecked", apiKeyError?: string, agentSdk: "available"\|"not_found" }` | 健康檢查 |

---

## WebSocket 事件

連線至：`ws://localhost:8000/ws/session/:sessionId`

所有事件皆以包含 `type` 辨識欄位的 JSON 格式，從伺服器流向用戶端。

### 工作階段生命週期

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `session_started` | `{ session_id }` | 工作階段已建立 |
| `planning_started` | -- | 元規劃器正在分解規格 |
| `plan_ready` | `{ tasks: Task[], agents: Agent[], explanation: string, deployment_target?: string }` | 任務 DAG 已就緒，準備執行 |
| `workspace_created` | `{ nugget_dir: string }` | 已建立 Nugget 工作區目錄 |
| `session_complete` | `{ summary }` | 建置已完成 |
| `error` | `{ message, recoverable: boolean }` | 發生錯誤 |

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
| `agent_status` | `{ agent: Agent }` | 代理狀態變更 (閒置/工作中/完成/錯誤/等待中) |
| `agent_message` | `{ from, to, content }` | 代理間的通訊 |
| `token_usage` | `{ agent_name, input_tokens, output_tokens, cost_usd }` | 每個代理的 Token 消耗量 |
| `budget_warning` | `{ total_tokens, max_budget, cost_usd }` | 達到 Token 預算門檻 |
| `minion_state_change` | `{ agent_name, old_status, new_status }` | 僕從狀態轉換 |
| `narrator_message` | `{ from, text, mood, related_task_id? }` | 講述者對建置事件的評論 |
| `permission_auto_resolved` | `{ task_id, permission_type, decision, reason }` | 代理權限請求已由策略自動解析 |

### 建置產物

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `commit_created` | `{ sha, message, agent_name, task_id, timestamp, files_changed }` | Git 提交已建立 |
| `test_result` | `{ test_name, passed: boolean, details }` | 個別測試結果 |
| `coverage_update` | `{ percentage, details?: CoverageReport }` | 程式碼覆蓋率報告 |

### 技能執行

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `skill_started` | `{ skill_id, skill_name }` | 技能計畫執行開始 |
| `skill_step` | `{ skill_id, step_id, step_type, status }` | 技能步驟開始/完成/失敗 |
| `skill_question` | `{ skill_id, step_id, questions: QuestionPayload[] }` | 技能詢問使用者問題 |
| `skill_output` | `{ skill_id, step_id, content }` | 技能步驟產生輸出 |
| `skill_completed` | `{ skill_id, result }` | 技能計畫已完成 |
| `skill_error` | `{ skill_id, message }` | 技能計畫失敗 |

### 部署

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `deploy_started` | `{ target }` | 部署階段開始 |
| `deploy_progress` | `{ step, progress: number }` | 部署進度 (0-100) |
| `deploy_checklist` | `{ rules: Array<{ name, prompt }> }` | 部署前規則檢查清單 |
| `deploy_complete` | `{ target, url? }` | 部署已完成 |
| `serial_data` | `{ line, timestamp }` | ESP32 序列埠監控輸出 |

### 使用者互動

| 事件 | 負載 | 說明 |
|-------|---------|-------------|
| `human_gate` | `{ task_id, question, context }` | 建置暫停，等待使用者批准 |
| `user_question` | `{ task_id, questions: QuestionPayload[] }` | 代理詢問使用者問題 |
| `teaching_moment` | `{ concept, headline, explanation, tell_me_more?, related_concepts? }` | 學習時刻呈現 |

**QuestionPayload**：
```typescript
{
  question: string;
  header: string;
  options: Array<{ label: string; description: string }>;
  multiSelect: boolean;
}
```

**講述者訊息情緒 (Moods)**：`excited` (興奮), `encouraging` (鼓勵), `concerned` (擔憂), `celebrating` (慶祝)

---

## 講述者系統 (Narrator System)

講述者透過 Claude Haiku 將原始建置事件轉化為適合兒童的評論。

### 觸發事件

這些建置事件會觸發講述者訊息：`task_started`, `task_completed`, `task_failed`, `agent_message`, `error`, `session_complete`。

### WebSocket 事件

```typescript
{
  type: "narrator_message";
  from: string;           // 講述者角色名稱
  text: string;           // 適合兒童的訊息 (最多 200 字)
  mood: string;           // "excited" | "encouraging" | "concerned" | "celebrating"
  related_task_id?: string;
}
```

### 配置

- `NARRATOR_MODEL` 環境變數可覆寫模型 (預設：`claude-haiku-4-5-20241022`)

### 去抖動 (Debounce)

`agent_output` 事件會按任務累積，並在 10 秒靜默期後進行轉譯。這可避免在代理快速輸出期間讓教學訊息淹沒 UI。

### 頻率限制

每個任務每 15 秒最多傳送 1 條講述者訊息。超過此限制的訊息將被默默捨棄。

---

## 門戶安全 (Portal Security)

### 指令允許清單

CLI 門戶在執行前會根據嚴格的允許清單驗證指令：

```
node, npx, python, python3, pip, pip3, mpremote, arduino-cli, esptool, git
```

任何不在清單中的指令都會被拒絕並傳回錯誤。

### 執行模型

`CliPortalAdapter.execute()` 使用 `execFile` (而非帶有 `shell: true` 的 `spawn`)。這可防止 Shell 注入，因為 `execFile` 會完全跳過 Shell -- 參數會直接傳遞給可執行檔案而不經過 Shell 解析。

### 序列埠門戶驗證

序列埠門戶在燒錄操作進行前會透過開發板偵測 (USB VID:PID 匹配) 進行驗證。這可確保目標裝置在嘗試寫入韌體前確實是 ESP32。

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
    visual: string | null;  // "fun_colorful" | "clean_simple" | "dark_techy" | "nature" | "space"
    personality: string | null;
  };
  agents: Array<{
    name: string;
    role: string;           // "builder" | "tester" | "reviewer" | "custom"
    persona: string;
  }>;
  hardware?: {
    target: string;         // "esp32"
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
    flow_hints?: Array<{
      type: "sequential" | "parallel";
      descriptions: string[];
    }>;
    iteration_conditions?: string[];
  };
  skills?: Array<{
    id: string;
    name: string;
    prompt: string;
    category: string;       // "agent" | "feature" | "style"
  }>;
  rules?: Array<{
    id: string;
    name: string;
    prompt: string;
    trigger: string;        // "always" | "on_task_complete" | "on_test_fail" | "before_deploy"
  }>;
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

## SkillSpec 結構

定義可重複使用的技能。簡單技能具備 `prompt`。複合技能則另外具備 `workspace` (流程編輯器的 Blockly JSON)。

```typescript
interface SkillSpec {
  id: string;               // 唯一的技能 ID (最多 200 字)
  name: string;             // 顯示名稱 (最多 200 字)
  prompt: string;           // 提示詞範本，支援 {{key}} 變數 (最多 5000 字)
  category: string;         // "agent" | "feature" | "style" | "composite"
  workspace?: Record<string, unknown>;  // Blockly 工作區 JSON (僅限複合技能)
}
```

---

## SkillPlan 結構

傳送至 `POST /api/skills/run`。代表要執行的步驟序列。

```typescript
interface SkillPlan {
  skillId: string;          // 正在執行的技能 ID (最多 200 字，選填)
  skillName: string;        // 顯示名稱 (最多 200 字)
  steps: SkillStep[];       // 有序步驟 (最多 50 個)
}
```

### SkillStep (根據 `type` 屬性的可辨識聯集)

所有步驟共享 `id: string` (最多 200 字)。6 種步驟類型：

| 類型 | 欄位 | 說明 |
|------|--------|-------------|
| `ask_user` | `question` (最多 2000), `header` (最多 200), `options: string[]` (最多 50 個項目), `storeAs` | 暫停執行，呈現選項給使用者。答案存儲在 context 的 `storeAs` 鍵下。 |
| `branch` | `contextKey`, `matchValue` (最多 500), `thenSteps: SkillStep[]` (最多 50, 遞迴) | 僅當 `context[contextKey] === matchValue` 時才執行 `thenSteps`。無 else 分支。 |
| `invoke_skill` | `skillId`, `storeAs` | 呼叫另一個技能。循環偵測 (最大深度 10)。結果存儲在 `storeAs` 下。 |
| `run_agent` | `prompt` (最多 5000), `storeAs` | 產生一個 Claude 代理。提示詞支援 `{{key}}` 範本。結果存儲在 `storeAs` 下。 |
| `set_context` | `key`, `value` (最多 5000) | 設定上下文變數。值支援 `{{key}}` 範本。 |
| `output` | `template` (最多 5000) | 產出最終的技能輸出。範本支援 `{{key}}` 語法。 |

### 上下文解析鏈 (Context Resolution Chain)

解析 `{{key}}` 時：
1. 檢查目前技能的 context 項目
2. 走訪父級 context (適用於巢狀 `invoke_skill` 呼叫)
3. 若未找到，傳回空字串

```typescript
interface SkillContext {
  entries: Record<string, string | string[]>;
  parentContext?: SkillContext;
}
```
