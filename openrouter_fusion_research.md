# OpenRouter Fusion 深度技術研究報告

> 研究日期：2026-06-20  
> 資料來源：OpenRouter Blog, API Docs, Server Tools Docs, Plugin Docs  
> 參考文章：[Surpassing Frontier Performance with Fusion](https://openrouter.ai/blog/announcements/fusion-beats-frontier/)

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

## 六、與 opencode Skill / Workflow 的對應分析

### 可實現的模式

利用 opencode 的 `task` 工具，可以近似模擬 Fusion 的工作流：

```
AI 判斷問題是否適合多角度分析
  → 同時啟動 2-3 個 Task subagent（不同系統指令視角）
  → 收集所有回應
  → AI 本體扮演 Judge，進行綜合分析
  → 產出最終答案
```

### 現有差異

| 面向 | OpenRouter Fusion | opencode Skill 方案 |
|------|-------------------|---------------------|
| 模型多樣性 | 不同架構模型（Claude/GPT/Gemini） | 同一模型（僅 prompt 差異） |
| 執行方式 | Server-side 平行 API call | Task 工具平行啟動 |
| Judge 模型 | 可獨立指定（如 Opus 4.8） | AI 本體自行判斷 |
| 遞迴保護 | x-openrouter-fusion-depth header | 需手動設計防護 |
| Tool 整合 | 內建 web_search + web_fetch | 依賴文件內已有的工具 |
| 延遲 | 2-3x 普通請求 | 取決於 subagent 數量與複雜度 |

### 設計建議

1. **Skill 方案（立即可行）**：定義一套引導提示，讓 AI 在遇到複雜問題時自動啟動多視角 subagent 並綜合結果
2. **Plugin 方案（需 opencode SDK 支援）**：如果 `task` 工具未來能指定不同模型，即可實現真正的多模型 fusion

---

## 七、參考資料

- [Fusion Blog Announcement](https://openrouter.ai/blog/announcements/fusion-beats-frontier/)
- [Fusion Server Tool Docs](https://openrouter.ai/docs/guides/features/server-tools/fusion)
- [Fusion Plugin Docs](https://openrouter.ai/docs/guides/features/plugins/fusion)
- [Fusion Router Docs](https://openrouter.ai/docs/guides/routing/routers/fusion-router)
- [DRACO Benchmark (arXiv)](https://arxiv.org/abs/2602.11685)
- [OpenRouter May Release Spotlight](https://openrouter.ai/blog/announcements/may-release-spotlight/)
- [Fusion Lab (interactive playground)](https://openrouter.ai/labs/fusion)
