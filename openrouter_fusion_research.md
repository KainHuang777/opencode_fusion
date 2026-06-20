# OpenRouter Fusion 深度技術研究報告

> 研究日期：2026-06-20（初版）、2026-06-21（深度 Fusion 分析更新）  
> 資料來源：OpenRouter Blog, API Docs, Server Tools Docs, Plugin Docs  
> 參考文章：[Surpassing Frontier Performance with Fusion](https://openrouter.ai/blog/announcements/fusion-beats-frontier/)  
> 分析方式：使用本專案 fusion-research skill 進行 4-panel 多模型協同分析（Judge: DeepSeek V4 Pro）

---

## 一、什麼是 OpenRouter Fusion？

Fusion 是 OpenRouter 在 2026 年 5 月推出的多模型協同推論系統。其核心思想是：**將同一個提示同時發送給多個不同架構的模型，由一個 Judge 模型分析所有回覆的共識與差異，最後由外層模型產出綜合性答案。**

測試結果顯示，Fusion 在 DRACO deep research benchmark（100 項任務、10 個領域）上超越了所有單一前沿模型，包括 Claude Fable 5 和 GPT-5.5。

### 核心分數

| 類型 | 模型組合 | 分數 |
|------|---------|------|
| Fusion | Fable 5 + GPT-5.5 (Judge: Opus 4.8) | **69.0%** |
| Fusion | Opus 4.8 + GPT-5.5 + Gemini 3.1 Pro (Judge: Opus 4.8) | **68.3%** |
| Fusion | Opus 4.8 + GPT-5.5 (Judge: Opus 4.8) | **67.6%** |
| Solo | Claude Fable 5 | 65.3% |
| Fusion | Gemini 3 Flash + Kimi K2.6 + DeepSeek V4 Pro (Judge: Opus 4.8) | **64.7%** |
| Solo | DeepSeek V4 Pro | 60.3% |
| Solo | GPT-5.5 | 60.0% |
| Solo | Claude Opus 4.8 | 58.8% |

---

## 二、系統架構

### 2.1 完整 Pipeline

```
User Request
  │
  ▼
Outer Model（接收提示，決定何時調用 Fusion）
  │
  ├── 簡單問題 → 直接回答（不觸發 Fusion）
  │
  └── 複雜/研究型問題 → 調用 openrouter:fusion tool
        │
        ▼
      Panel（1~8 個模型，平行執行）
        │   每個模型附帶 web_search + web_fetch
        │   各自獨立產生回應
        │
        ▼
      Judge（分析模型）
        │   接收所有 panel 回應
        │   產出結構化 JSON：
        │   ├── consensus（共識）
        │   ├── contradictions（矛盾點）
        │   ├── partial_coverage（部分覆蓋）
        │   ├── unique_insights（獨特見解）
        │   └── blind_spots（盲點）
        │
        ▼
      Outer Model 接收 Analysis JSON
        │
        ▼
      Final Answer（綜合後的最終輸出）
```

### 2.2 三種接入方式

| 方式 | 說明 | 適用場景 |
|------|------|---------|
| **Model Slug** | 直接指定 `model: "openrouter/fusion"`，自動注入預設 panel | 最簡單，一鍵切換 |
| **Server Tool** | 在 tools 陣列中加入 `{ type: "openrouter:fusion" }`，由模型自行決定何時調用 | 最大控制權，可與其他 tools 搭配 |
| **Plugin** | 在 plugins 陣列中配置 `{ id: "fusion", ... }`，可自訂 panel 組合與 judge | 介於兩者之間，結構化配置 |

三種方式共用同一套後端 pipeline。

---

## 三、技術細節

### 3.1 Panel 配置參數

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `analysis_models` | Quality preset（Opus + GPT + Gemini） | Panel 模型列表，1~8 個 |
| `model` | 同外層模型 | Judge 模型 |
| `max_tool_calls` | 8 | Panel/judge 的 tool call 上限（1~16） |
| `max_completion_tokens` | Provider 預設 | 輸出 token 上限 |
| `reasoning` | Provider 預設 | 推理配置（effort / max_tokens） |
| `temperature` | Provider 預設 | 採樣溫度 (0~2) |

### 3.2 Judge 輸出格式

成功時回傳：
```json
{
  "status": "ok",
  "analysis": {
    "consensus": ["所有或大部分 panel 模型同意的點"],
    "contradictions": [
      { "topic": "...", "stances": [{ "model": "...", "stance": "..." }] }
    ],
    "partial_coverage": [
      { "models": ["..."], "point": "僅部分模型涵蓋此面向" }
    ],
    "unique_insights": [
      { "model": "...", "insight": "僅單一模型提出的觀點" }
    ],
    "blind_spots": ["所有 panel 模型都未觸及的主題"]
  },
  "responses": [
    { "model": "anthropic/claude-opus-4.5", "content": "..." }
  ]
}
```

### 3.3 優雅降級機制

| 情境 | 行為 |
|------|------|
| Judge 失敗但 panel 成功 | 回傳 `status: "ok"`，省略 analysis，保留 raw responses |
| 部分 panel 失敗 | 回傳 `status: "ok"`，附加 `failed_models[]` |
| 全部 panel 失敗 | 回傳 `status: "error"`，包含 typed `failure_reason` |
| 同次請求中第二次調用 | 被 `fusion_invocation_capped` 拒絕（單層遞迴保護） |

### 3.4 Preset 系統

OpenRouter 提供預設 panel 組合，免除手動選模型：

| Preset | 用途 |
|--------|------|
| `general-high` | 頂級通用 panel（最強模型組合） |
| `general-budget` | 平價 panel + 前沿 judge（成本約 50%） |

---

## 四、關鍵發現

### 4.1 Self-Fusion 效應

將 Opus 4.8 與自己配對（Opus 4.8 × 2，同樣由 Opus 4.8 做 judge）：
- Solo Opus 4.8：**58.8%**
- Self-Fusion Opus 4.8：**65.5%（+6.7 分）**

這表明 **synthesis 步驟本身就帶來顯著提升**——同一模型跑兩次會產生不同的推理路徑、不同的 tool call 選擇、不同的來源引用，而 Judge 的融合分析能從中萃取出更好的結果。

### 4.2 模型多樣性的價值

不同架構模型（Claude / GPT / Gemini / DeepSeek / Kimi）各有其推理風格與擅長領域，組合在一起能有效涵蓋彼此的盲點。

### 4.3 性價比突破

平價 panel（Gemini 3 Flash + Kimi K2.6 + DeepSeek V4 Pro）+ Opus 4.8 Judge：
- 分數：64.7%（接近 Fable 5 的 65.3%）
- 成本：約 Fable 5 的 **50%**

---

## 五、限制與注意事項

1. **延遲成本**：Fusion 調用時約為普通請求的 2~3 倍時間（等待多個模型 + Judge 分析）
2. **非編碼替代品**：Fusion 不適合作為 coding 模型的直接替代——建議用於架構決策或研究類問題，常規編碼由編碼模型直接處理
3. **測試局限**：DRACO 僅涵蓋文本、英文任務，且靜態 task set 可能無法全面泛化至未來應用場景
4. **Judge 偏差**：絕對分數會隨 Judge 模型選擇而變化（論文報吿 10~25 分差異），但相對排名穩定
5. **不包含長時序任務**：Fable 5 這類擅長長時間任務的模型，其優勢不在 DRACO 的評測範圍內

---

## 六、opencode Fusion Skill 實作現況

### 6.1 架構總覽

本專案已實作完整的多模型 Fusion Skill 系統，運行於 opencode 編輯器內：

```
opencode.jsonc (中央配置，預設模型 DeepSeek V4 Pro)
  ├── Skills (2 個): fusion-research, fiction-editor
  ├── Research Panels (8 個): fusion-deepseek, fusion-kimi, fusion-qwen,
  │     fusion-glm, fusion-gemini, fusion-skyunion, fusion-budget-ds, fusion-budget-mimo
  └── Fiction Panels (3 個): fusion-fiction-plot, fusion-fiction-character, fusion-fiction-prose

Pipeline:
  1. Triage (AI 判斷問題是否需要 Fusion)
  2. Prompt Design (為每個 panel 設計不同視角的 prompt)
  3. Parallel Dispatch (task() 工具平行啟動 2-4 panel subagent)
  4. Judge Synthesis (主 AI 綜合分析：共識/分歧/獨特見解/盲點)
  5. Self-Evaluation (信心水準)
```

### 6.2 模型多樣性（6 個 Provider、8 個不同架構）

| Agent | Model | Provider | 用途 |
|-------|-------|----------|------|
| fusion-deepseek | deepseek-v4-pro | DeepSeek | 技術深度、程式推理 |
| fusion-kimi | kimi-k2.7-code | Moonshot | 程式架構、邏輯流 |
| fusion-qwen | qwen3.7-plus | Alibaba | 綜合分析、廣域脈絡 |
| fusion-glm | glm-5.2 | Zhipu | 創意思考、替代視角 |
| fusion-gemini | gemini-3.5-flash | Google | 跨域觀點、實務約束 |
| fusion-skyunion | claude-haiku-4.5 | Anthropic (第三方) | 細緻權衡、安全倫理 |
| fusion-budget-ds | deepseek-v4-flash | DeepSeek | 快速平價分析 |
| fusion-budget-mimo | mimo-v2.5 | MiMo | 平價廣域覆蓋 |

### 6.3 現有差異（更新版）

| 面向 | OpenRouter Fusion | opencode Fusion Skill |
|------|-------------------|-----------------------|
| 模型多樣性 | 動態模型池（數百個模型） | 靜態 agent 註冊（8 個模型，6 個 provider） |
| 執行方式 | Server-side 平行 API call | Task 工具平行啟動 subagent |
| Judge 模型 | 可獨立指定任意模型 | 主 AI (DeepSeek V4 Pro)，強制 bias 避免規則 |
| Judge 輸出 | 強制結構化 JSON | 手動 Markdown 模板 |
| 遞迴保護 | x-openrouter-fusion-depth header | 未實作 |
| 優雅降級 | 原生型別化（failed_models[] 等） | 未自動化（skill 文件描述但無程式碼強制） |
| Tool 整合 | Panel 內建 web_search + web_fetch | Agent 設定 webfetch/websearch allow |
| Preset 系統 | general-high / general-budget | 手動定義 4 種 tier（Free/Budget/Flagship/Self） |
| 領域專用 | 無（通用融合） | 有（research + fiction-editor 雙 pipeline） |
| 可解釋性 | 對使用者透明（黑箱） | 完全透明（用戶可看每個 panel 推理過程） |
| 本地整合 | 無 | 可讀取工作區檔案、理解專案上下文 |

### 6.4 Skill 方案的四種 Tier

| Tier | Panel 組合 | 適用場景 |
|------|-----------|---------|
| **Free (4-panel)** | Kimi + Qwen + Gemini + Claude | 預設，最大化模型多樣性 |
| **Budget (3-panel)** | Kimi + MiMo + DeepSeek Flash | 成本敏感場景 |
| **Flagship (3-panel)** | Kimi + Qwen + GLM | 最高品質需求 |
| **Self-Fusion (2-panel)** | DeepSeek × 2 | 同模型多視角（+5.2 預期增益） |

### 6.5 Judge Bias 規則

Judge 為 DeepSeek V4 Pro，所有 DeepSeek 系列 panel 預設排除（Self-Fusion 除外）。此規則確保 Judge 不會系統性偏好與自己同架構的模型輸出。

### 6.6 第三方 API 整合經驗（Anthropic Claude）

透過 OpenAI-compatible 第三方 API 接入 Claude Haiku 4.5，補足了現有 panel 陣容中唯一缺少的 Anthropic 架構：

| 面向 | 細節 |
|------|------|
| 提供者 | 第三方 API（OpenAI-compatible endpoint） |
| 模型 | Claude Haiku 4.5 |
| 架構貢獻 | Anthropic — 與 Moonshot、Alibaba、Google 形成四種完全不同的模型架構 |
| 特性優勢 | 細膩權衡推理、安全倫理意識、二階效應分析 |
| 穩定度 | ✅ 穩定可用（經多輪測試驗證） |
| 成本 | 依第三方 API 定價 |

**架構多樣性現狀（4/4 完成）：**

```
Moonshot (Kimi K2.7)    — 程式架構視角
Alibaba (Qwen3.7 Plus)  — 綜合廣度視角
Google (Gemini 3.5 Flash) — 跨域替代視角（free tier，受每日配額限制）
Anthropic (Claude Haiku 4.5) — 細膩權衡視角（第三方API）
```

四種完全不同的模型架構平行分析，加上 DeepSeek V4 Pro 作為 Judge，最大程度覆蓋彼此盲點。這是目前 opencode 生態中架構多樣性最完整的 Fusion 實作方案。

---

## 七、參考資料

- [Fusion Blog Announcement](https://openrouter.ai/blog/announcements/fusion-beats-frontier/)
- [Fusion Server Tool Docs](https://openrouter.ai/docs/guides/features/server-tools/fusion)
- [Fusion Plugin Docs](https://openrouter.ai/docs/guides/features/plugins/fusion)
- [Fusion Router Docs](https://openrouter.ai/docs/guides/routing/routers/fusion-router)
- [DRACO Benchmark (arXiv)](https://arxiv.org/abs/2602.11685)
- [OpenRouter May Release Spotlight](https://openrouter.ai/blog/announcements/may-release-spotlight/)
- [Fusion Lab (interactive playground)](https://openrouter.ai/labs/fusion)

---

## 八、Fusion Research 深度分析報告（2026-06-21）

> 使用本專案 fusion-research skill 對「opencode Fusion Skill 改進空間與本質差異」進行 4-panel 多模型協同分析。  
> Panel: Kimi K2.7 (技術架構) + Qwen3.7 Plus (綜合策略) + Gemini 3.5 Flash (替代視角) + Claude Haiku 4.5 (細緻差異)  
> Judge: DeepSeek V4 Pro  
> 結果: 3/4 panel 成功（Kimi, Qwen, Claude 第三方API）、1/4 因 free tier 限流失敗（Gemini）——Partial failure 的真實案例

### 8.1 三層級差異分析

#### 第一層：架構層級（不可跨越的差異）

| 面向 | OpenRouter Fusion | opencode Fusion Skill | 是否需要消除 |
|------|-------------------|-----------------------|-------------|
| 運行層級 | Server-side API 閘道 | Client-side IDE 編輯器 | **否** — 我們的優勢在本地整合 |
| 模型存取 | 動態模型池（數百個） | 靜態 agent 註冊（8 個） | 中期可增加 agent 數量 |
| 輸出格式 | 強制結構化 JSON | 手動 Markdown 模板 | 部分 — 可選 JSON mode |
| 錯誤處理 | 原生型別化降級 | 未自動化 | **是** — P0 優先級 |
| 遞迴保護 | 原生 header | 未實作 | **是** — P1 優先級 |

**結論：架構層級差異是本質性的，不需要消除。** Skill-based 方案在本地工作流整合、可解釋性、可定制性上有 OpenRouter 無法觸及的優勢。

#### 第二層：功能層級（可改進的差距）

| 缺失功能 | 優先級 | 影響範圍 | 預期效果 |
|---------|--------|---------|---------|
| 結構化 partial failure / failed_models[] | **P0** | 可靠性 | 避免使用者在 panel 失敗時得到不完整資訊 |
| Judge fail fallback（保留 raw responses） | **P0** | 韌性 | 即使 Judge 失敗，panel 的原始回應仍有價值 |
| 遞迴保護（fusion depth tracking） | **P1** | 安全性 | 避免 panel agent 意外觸發新的 fusion 造成成本爆炸 |
| 結構化 JSON Judge 輸出（可選） | **P1** | 一致性 | 便於下游 agent/UI 消費；保留 Markdown 作為 fallback |
| Web 工具權限顯式配置 | **P1** | 功能完整性 | 確保 panel 能真正使用 web_search/web_fetch |
| Timeout/retry 策略定義 | **P1** | 穩定性 | 避免 panel 逾時後流程卡住 |
| Progressive Fusion（漸進式啟動） | **P2** | 成本效益 | 簡單問題不浪費 panel 資源 |
| Fusion Memory（跨次學習） | **P2** | 智慧化 | 記錄最佳模型組合，隨時間進化 |

#### 第三層：策略層級（方向性選擇）

1. **建立量化評估機制（P0 策略）**
   - 沒有數據就無法證明 fusion 的價值。需要建立類似 DRACO 的內部 benchmark，定期測試 fusion vs solo 在不同場景的表現。

2. **擴展領域專用 skill（P1 策略）**
   - fiction-editor 證明了領域深度定制是護城河。優先新增 Code Review / Architecture Decision skill。

3. **保持「可解釋性」核心價值（P0 策略）**
   - 與 OpenRouter 的「黑箱優化」不同，「透明推理」是用戶選擇我們的關鍵原因。不要為了結構化而犧牲可讀性。

4. **抽取 Fusion Engine 核心（中期策略，6-18 個月）**
   - 將 fusion orchestration 邏輯抽取為可獨立使用的模組，降低對 opencode task() API 的依賴。

### 8.2 共識要點

所有 panel 高度一致的觀點：

- **Skill-based 方案與 OpenRouter 是互補關係，非競爭關係**——我們解決「特定場景深度」，他們解決「通用調用便利」
- **最大缺口是缺乏量化評估**——沒有 benchmark 就無法證明 fusion 的實際提升
- **錯誤處理與優雅降級是 P0**——此次分析中 2/4 panel 失敗本身就是證據
- **領域專用 pipeline 是最大競爭優勢**——fiction-editor 的 V3→V4 改進數據證明此方向可行
- **應新增 Code Review / Architecture Decision skill**——覆蓋開發者最高頻的日常場景

### 8.3 分歧要點

| 議題 | 技術視角 (Kimi) | 策略視角 (Qwen) | Judge 判斷 |
|------|----------------|----------------|-----------|
| JSON 輸出優先級 | P1，中期導入 | 未強調（更重可解釋性） | P1 正確，但必須保留 Markdown fallback |
| 架構演進路徑 | 遷移至 SDK/plugin | 深耕 skill 生態 → 中期抽取引擎 | 兩者不矛盾，短期深耕 skill，中期抽取引擎 |
| 遞迴保護 | P1 明確要求 | 未討論 | P1 正確，成本爆炸風險真實存在 |

### 8.4 盲點（本次分析未觸及的主題）

由於 Gemini（free tier 配額限制）panel 失敗，以下主題未能充分覆蓋：

1. **Self-Fusion 的策略定位與最佳使用場景**
2. **多語言場景（中/英）下 fusion 效果的差異**
3. **Panel 間交互模式**（如 panel A 輸出作為 panel B 的 context）
4. **二階效應**：使用者對 fusion 的過度信任、假共識問題、資訊密度下降
5. **安全與對齊**：多模型對安全問題的不同答案處理機制

Claude (第三方API) panel 成功參與本次分析，提供 Anthropic 架構視角，補足了安全倫理與二階效應的覆蓋。後續建議解決 Gemini free tier 配額限制，或替換為同等架構多樣性的替代方案。

### 8.5 改進行動路線圖

```
Phase 1 (立即 - 1 週內):
  ├── P0: 定義並實作 failed_models[] 結構化追蹤
  ├── P0: 實作 Judge fail fallback（保留 raw responses）
  └── P0: 建立內部評估框架（至少 10 題 benchmark）

Phase 2 (短期 - 1 個月內):
  ├── P1: 加入遞迴保護機制
  ├── P1: 引入可選 JSON Judge 輸出模式
  ├── P1: 驗證並顯式配置 web 工具權限
  ├── P1: 定義 timeout/retry 策略
  └── P1: 設計 Code Review / Architecture Decision skill

Phase 3 (中期 - 3 個月內):
  ├── P2: Progressive Fusion（漸進式啟動）
  ├── P2: Fusion Memory（記錄最佳模型組合）
  └── P2: 抽取 Fusion Engine 核心模組

Phase 4 (長期 - 6-18 個月):
  ├── 定義 Fusion-as-a-Protocol 開放標準
  ├── 實現多語言場景的專用 fusion 配置
  └── 建立社群貢獻的 skill 生態系統
```
