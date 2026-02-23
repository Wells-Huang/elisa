# 積木參考

Elisa 積木調色盤完整指南。積木在畫布上組合後會產出 [NuggetSpec](api-reference.zh-tw.md#nuggetspec-結構-schema)，用以驅動建置工作。

類別：[目標](#目標) | [需求](#需求) | [風格](#風格) | [技能](#技能) | [規則](#規則) | [技能流程](#技能流程) | [門戶](#門戶) | [僕從](#僕從) | [流程](#流程) | [部署](#部署)

---

## 目目標

定義您要構建的內容。每個專案至少需要一個目標積木。

| 積木 | 欄位 | NuggetSpec 產出 |
|-------|--------|--------------------|
| **Nugget 目標** | `GOAL_TEXT` (文字輸入) | `nugget.goal`, `nugget.description` |
| **Nugget 範本** | `TEMPLATE_TYPE` (下拉選單) | `nugget.type` |

**範本類型**：`game` (遊戲), `website` (網站), `hardware` (硬體), `story` (故事), `tool` (工具)

---

## 需求

描述專案應該具備的功能。

| 積木 | 欄位 | NuggetSpec 產出 |
|-------|--------|--------------------|
| **功能 (Feature)** | `FEATURE_TEXT` (文字) | `requirements[]` 且 `type: "feature"` |
| **限制 (Constraint)** | `CONSTRAINT_TEXT` (文字) | `requirements[]` 且 `type: "constraint"` |
| **當...就... (When/Then)** | `TRIGGER_TEXT`, `ACTION_TEXT` (文字) | `requirements[]` 且 `type: "when_then"` |
| **包含資料 (Has Data)** | `DATA_TEXT` (文字) | `requirements[]` 且 `type: "data"` |

**範例**：帶有「支援多人連線」的「功能」積木會產出 `{ type: "feature", description: "支援多人連線" }`。

---

## 風格

控制產出的外觀與個性。

| 積木 | 欄位 | NuggetSpec 產出 |
|-------|--------|--------------------|
| **看起來像 (Look Like)** | `STYLE_PRESET` (下拉選單) | `style.visual` |
| **個性 (Personality)** | `PERSONALITY_TEXT` (文字) | `style.personality` |

**風格預設值**：`fun_colorful` (趣味豐富), `clean_simple` (簡潔乾淨), `dark_techy` (科技深色), `nature` (自然), `space` (太空)

---

## 技能

可重複使用的提示詞片段，用以擴展代理的能力。在技能彈窗（側欄的扳手圖示）中建立。

| 積木 | 欄位 | NuggetSpec 產出 |
|-------|--------|--------------------|
| **使用技能** | `SKILL_ID` (下拉選單，動態產生) | `skills[]` |

每個技能都有名稱、提示詞和類別（`agent`, `feature` 或 `style`）。簡單技能包含提示詞範本；複合技能則使用流程編輯器（請參閱[技能流程](#技能流程)）。

---

## 規則

在建置過程中自動觸發的護欄。在規則彈窗（側欄的盾牌圖示）中建立。

| 積木 | 欄位 | NuggetSpec 產出 |
|-------|--------|--------------------|
| **套用規則** | `RULE_ID` (下拉選單，動態產生) | `rules[]` |

每個規則都有名稱、提示詞和觸發條件：`always` (總是), `on_task_complete` (任務完成時), `on_test_fail` (測試失敗時) 或 `before_deploy` (部署前)。

---

## 技能流程

用於複合技能的視覺化流程編輯器。將多個步驟鏈接起來以建立多步驟的代理工作流。在技能彈窗內開啟流程編輯器。

所有 7 個流程積木從「技能流程 (Skill Flow)」開始，由上而下連接。

| 積木 | 欄位 | 行為 |
|-------|--------|----------|
| **技能流程** | *(無)* | 入口點。必須是每個流程中的第一個積木。 |
| **詢問使用者** | `QUESTION` (文字), `HEADER` (標題文字), `OPTIONS` (逗點分隔文字), `STORE_AS` (鍵名) | 暫停執行並呈現選項給使用者。將所選答案以 `STORE_AS` 存儲於 context 中。 |
| **If** | `CONTEXT_KEY` (鍵名), `MATCH_VALUE` (文字), `THEN_BLOCKS` (敘述插槽) | 根據 context 值進行分支。僅當 `CONTEXT_KEY` 的值等於 `MATCH_VALUE` 時，才執行 `THEN_BLOCKS`。無 else 分支 —— 每種情況請使用多個 If 積木。 |
| **執行技能** | `SKILL_ID` (下拉選單), `STORE_AS` (鍵名) | 根據 ID 調用另一個技能。將技能的產出存儲於 context 中。支援最高 10 層的巢狀調用，具備循環偵測。 |
| **執行代理** | `PROMPT` (多行文字), `STORE_AS` (鍵名) | 帶著給定的提示詞範本產生一個 Claude 代理。將代理的結果摘要存儲於 context 中。 |
| **設定上下文** | `KEY` (鍵名), `VALUE` (文字) | 設定上下文變數。可用於組合或轉換值。 |
| **輸出** | `TEMPLATE` (文字) | 產出技能流程的最終結果。終端積木 (無下方接口)。 |

### 上下文變數

在任何文字欄位中使用 `{{key}}` 語法來引用 context 值：

```
詢問使用者 -> 存儲為 "topic"
執行代理 -> 提示詞 "構建一個 {{topic}} 應用程式" -> 存儲為 "result"
輸出 -> 範本 "已完成：{{result}}"
```

上下文解析會先搜尋目前上下文，然後是父級上下文（針對巢狀技能調用）。

### 分支行為

`If` 積木沒有 else 分支。要處理多種情況，請鏈接多個 `If` 積木：

```
If answer 等於 "遊戲"    -> [執行代理：構建一個遊戲]
If answer 等於 "網站"    -> [執行代理：構建一個網站]
```

每個積木以第一個匹配項優先。所有 `If` 積木均獨立評估（並非互斥）。

---

## 門戶 (Portals)

連接外部硬體與服務。門戶下拉選單會根據配置的門戶動態產生。

| 積木 | 欄位 | NuggetSpec 產出 |
|-------|--------|--------------------|
| **告訴 (Tell)** | `PORTAL_ID` (下拉選單), `CAPABILITY_ID` (下拉選單，過濾為 actions), 以及動態的 `PARAM_*` 欄位 | `portals[]` 且 `command: "tell"` |
| **當...時 (When)** | `PORTAL_ID` (下拉選單), `CAPABILITY_ID` (下拉選單，過濾為 events), `ACTION_BLOCKS` (敘述插槽), 以及動態的 `PARAM_*` 欄位 | `portals[]` 且 `command: "when"` |
| **詢問 (Ask)** | `PORTAL_ID` (下拉選單), `CAPABILITY_ID` (下拉選單，過濾為 queries), 以及動態的 `PARAM_*` 欄位 | `portals[]` 且 `command: "ask"` |

**告訴 (Tell)** 向門戶發送單次指令（例如，「告訴 LED 燈條 set_color」）。**當...時 (When)** 對門戶事件做出反應（例如，「當按鈕按下時，執行...」）。**詢問 (Ask)** 向門戶查詢資料（例如，「詢問感測器目前溫度」）。參數欄位會根據選定的能力動態新增。

---

## 僕從 (Minions)

配置將構建您專案的 AI 僕從。若未放置僕從積木，將使用預設值。

| 積木 | 欄位 | 角色 | NuggetSpec 產出 |
|-------|--------|------|--------------------|
| **構建僕從** | `AGENT_NAME`, `AGENT_PERSONA` (文字) | `builder` | `agents[]` |
| **測試僕從** | `AGENT_NAME`, `AGENT_PERSONA` (文字) | `tester` | `agents[]` |
| **評閱僕從** | `AGENT_NAME`, `AGENT_PERSONA` (文字) | `reviewer` | `agents[]` |
| **自定義僕從** | `AGENT_NAME`, `AGENT_PERSONA` (文字) | `custom` | `agents[]` |

個性 (Persona) 欄位會塑造僕從的行為。範例：名為「SpeedBot」的構建僕從，個性為「寫出最精簡、快速的程式碼」，將會依此獲得提示詞。

---

## 流程 (Flow)

控制執行順序。這些是容器積木，內部可以放置其他積木。

| 積木 | 輸入 | NuggetSpec 產出 |
|-------|--------|--------------------|
| **先...然後... (First/Then)** | `FIRST_BLOCKS`, `THEN_BLOCKS` (敘述插槽) | `workflow.flow_hints[]` 且 `type: "sequential"` |
| **同時進行 (At Same Time)** | `PARALLEL_BLOCKS` (敘述插槽) | `workflow.flow_hints[]` 且 `type: "parallel"` |
| **持續改進 (Keep Improving)** | `CONDITION_TEXT` (文字) | `workflow.iteration_conditions[]` |
| **與我確認 (Check With Me)** | `GATE_DESCRIPTION` (文字) | `workflow.human_gates[]` |
| **每隔一巡 (Timer Every)** | `INTERVAL` (數字，預設 5), `ACTION_BLOCKS` (敘述插槽) | `workflow.timers[]` |

**先...然後... (First/Then)** 在執行第二插槽的積木前會先執行第一插槽的積木。**同時進行 (At Same Time)** 會同時運行內含的積木。**持續改進 (Keep Improving)** 迴圈執行直到滿足條件。**與我確認 (Check With Me)** 會暫停建置並詢問使用者是否批准。**每隔一巡 (Timer Every)** 按固定週期重複執行內含積木。

---

## 部署 (Deploy)

選擇建置完成後的專案要部署到何處。

| 積木 | NuggetSpec 產出 |
|-------|--------------------|
| **部署到網頁** | `deployment.target: "web"` |
| **部署到 ESP32** | `deployment.target: "esp32"` |
| **兩者皆部署** | `deployment.target: "both"` |

若未放置部署積木，預設為 `"preview"`。

---

## 組合範例

一個簡單的遊戲專案可能會使用：

1. **Nugget 目標**："一個小行星遊戲"
2. **Nugget 範本**：`game`
3. **功能**："三條命與分數計算器"
4. **功能**："難度隨波次增加"
5. **看起來像**：`space`
6. **構建僕從**：名稱 "GameDev"，個性 "寫出乾淨的 HTML5 canvas 遊戲"
7. **測試僕從**：名稱 "QA"，個性 "徹底測試邊緣案例"
8. **先...然後...**：第一插槽放置構建僕從，第二插槽放置測試僕從
9. **與我確認**："部署前先檢閱遊戲"
10. **部署到網頁**

這會產出一個具備循序流程、部署前有人類閘門、兩個僕從以及網頁部署目標的 NuggetSpec。
