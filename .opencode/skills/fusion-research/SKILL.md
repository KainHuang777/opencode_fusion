---
name: fusion-research
description: Use ONLY when the user asks complex research, architecture design, multi-faceted analysis, or strategic decision questions. Implements OpenRouter Fusion-style multi-model collaborative reasoning. Triggers on keywords: research, analyze deeply, compare models, architecture decision, multi-perspective, pros and cons, evaluate options, 分析, 研究, 比較, 評估. Do NOT use for simple coding, file operations, or single-answer questions.
---

# Fusion Research - Multi-Model Collaborative Analysis

This skill implements a multi-model "Fusion" pipeline inspired by OpenRouter Fusion, adapted for opencode's agent system. When a complex question is detected, dispatch it to multiple panel agents (each with a different model architecture) in parallel, then act as Judge to synthesize the results.

> **適用環境**：本版本（`.opencode/skills/`）使用**不同架構模型**作為 panel，需要多模型 token 環境，盲點互補效果最佳。
> 若你在 Antigravity 等**僅單一底層模型**的 IDE 運行，請改用 `.agents/skills/fusion-research`（單模型 Self-Fusion 版）。

## Core Principle

**不同架構的模型有不同的推理風格與擅長領域，同時發問、平行收集、綜合裁判，能有效涵蓋彼此的盲點。**

Reference research data (OpenRouter Fusion DRACO benchmark):
- Solo DeepSeek V4 Pro: 60.3%
- Self-Fusion (Opus 4.8 ×2, same model as Judge): 58.8% → 65.5% (+6.7 分)
- Multi-model Fusion (3 diverse models): 64.7% ~ 69.0%
- Best Solo (Claude Fable 5): 65.3%

> 註：Self-Fusion（同模型 ×2）是 OpenRouter 原始研究中正式存在的合法組合，並非禁忌。優先順序為「不同架構模型 > 同模型 ×2 > 單一模型」。上述 +6.7 分為 DRACO 上 Opus 4.8 的實測值。

### ⚠️ Critical Rule: Judge Bias Avoidance

**Judge 必須與所有 Panel 使用不同架構的模型**，否則會產生系統性偏差（模型傾向認同自己的輸出，即使提示詞不同）。

OpenRouter Fusion 研究報告指出 Judge 偏差會造成 10~25 分的絕對分數差異。

- 當 Judge 為 DeepSeek V4 Pro 時 → Panel 不得包含 DeepSeek 系列模型
- 當 Judge 為 GLM-5.2 時 → Panel 不得包含 GLM 系列模型
- **Self-Fusion** 是合法的例外（同模型做 Judge + Panel，DRACO 上 Opus 4.8 達 +6.7 分）。這是因為合成步驟本身新增價值，但仍需意識到存在 Judge 偏差。它是「無多模型環境時的次優選擇」，仍優於單一模型直接回答

## Activation Decision

ALWAYS evaluate the user's question against this triage checklist BEFORE answering:

### Progressive Triage (先做輕量判斷，再決定啟動規模)

在決定是否啟動 Fusion 之前，先快速評估問題的**複雜度 × 不確定性**：

1. **直接回答**（不啟動 Fusion）：
   - Simple coding task (write a function, fix a bug)
   - File read/write operations
   - Single-fact lookup
   - Short, unambiguous question with a clear correct answer
   - User explicitly requests a single model answer

2. **啟動 Fusion**（問題需同時滿足：多面向 + 高不確定性 + 不同視角有增益）：
   - Research/deep analysis question
   - Architecture or design decision with real trade-offs
   - Multi-faceted comparison or evaluation (A vs B with no obvious winner)
   - Strategic planning with competing priorities
   - 使用者用中文問複雜問題，且問題無單一標準答案

> **防止誤觸發**：「分析」、「研究」、「比較」等詞單獨出現不足以啟動 Fusion。
> 必須確認問題確實有多個合理視角，且增加 panel 能帶來實質增益。
> 例：「比較兩個函數的效能」→ 直接回答（有客觀答案）；「比較兩種架構的長期維護成本」→ 啟動 Fusion（視角依賴多）。

## Workflow

### Phase 1: Triage & Prompt Design

Analyze the user's question and craft panel-specific prompts:
1. Extract the core question
2. Design 2-3 perspective-specific prompts (each panel gets a different angle)
3. Each prompt should be self-contained and include the original question

**Prompt design guidelines (example with Judge = DeepSeek V4 Pro):**
- Panel 1 (Kimi/Technical): Focus on technical accuracy, data, logic
- Panel 2 (Qwen/Broad): Focus on comprehensiveness, context, practical implications
- Panel 3 (GLM/Alternative): Focus on creative angles, edge cases, counterarguments

**Always verify Judge != any Panel model.** Refer to the Judge Bias rule above.

### Phase 2: Panel Dispatch (Parallel)

**AUTO MODE** — if fusion agents are configured in opencode.jsonc:
- Launch 2-3 panel subagents IN PARALLEL via the `task` tool
- Available agent types: `fusion-deepseek`, `fusion-kimi`, `fusion-qwen`, `fusion-glm`, `fusion-minimax`, `fusion-mimo`
- Each agent has a different opencode-go model pre-assigned
- Use `task` tool with `subagent_type` matching the agent name

**MANUAL MODE** — if fusion agents are NOT configured:
- Inform the user clearly:
  1. "切換到模型 X，輸入以下指令："
  2. Show the exact prompt for each model
  3. "完成後將所有回應貼回來，我進行綜合裁判"
- Wait for user to provide all responses before proceeding to Phase 3

### Phase 2.5: 失敗處理 (Failure Handling)

收到所有 panel 回應後，先檢查：

- **部分 panel 失敗**（超時、API 限流、錯誤）：
  - 在報告中明確標記 `[FAILED: 模型名 — 原因]`
  - 以剩餘正常 panel 繼續進行 Judge Synthesis
  - 在最終報告開頭提示：「本次 X/N panel 成功」
- **全部 panel 失敗**：
  - 告知使用者並說明原因，建議改用 Budget 或直接回答
- **Judge 本身失敗**：
  - 不丟失已產出的 panel 結果
  - 直接將各 panel 原始回應整理後呈現給使用者，附上說明：「Judge 合成失敗，以下為各 panel 原始分析，請自行參考」

After collecting all panel responses, produce a structured synthesis:

```
## 🔬 Fusion 綜合分析報告

### 📊 參與模型
| 模型 | 視角 | 摘要 |
|------|------|------|
| Model A | 技術分析 | ... |
| Model B | 綜合評估 | ... |
| Model C | 創意視角 | ... |

### ✅ 共識 (Consensus)
所有或大部分模型同意的觀點：

### ⚠️ 分歧 (Contradictions)
模型之間存在矛盾的觀點：
| 議題 | Model A 立場 | Model B 立場 |
|------|-------------|-------------|

### 💡 獨特見解 (Unique Insights)
僅單一模型提出的有價值觀點：
- [Model X]: ...
- [Model Y]: ...

### 🔻 盲點 (Blind Spots)
所有模型都未觸及的重要面向：

### 🎯 綜合結論 (Final Answer)
綜合以上分析的最終建議：
```

### Phase 4: Self-Evaluation

After producing the final answer, briefly self-evaluate:
- Confidence level (high/medium/low)
- Key uncertainties remaining
- Suggested follow-up if any

## Panel Agent Reference

### Available Agent Types (when configured):

| Agent Name | Model | Best For | Approx Cost | Judge Compat |
|-----------|-------|----------|-------------|-------------|
| `fusion-deepseek` | opencode-go/deepseek-v4-pro | Technical depth, code reasoning | $1.74/$3.48 per 1M | ⚠️ Conflicts if Judge = DeepSeek |
| `fusion-kimi` | opencode-go/kimi-k2.7-code | Code architecture, logic flow | $0.95/$4.00 per 1M | ✅ All judges |
| `fusion-qwen` | opencode-go/qwen3.7-plus | Comprehensive analysis, broad context | $0.40/$1.60 per 1M | ✅ All judges |
| `fusion-glm` | opencode-go/glm-5.2 | Creative thinking, alternative angles | $1.40/$4.40 per 1M | ✅ All judges |
| `fusion-gemini` | google/gemini-3.5-flash | Google diversity, alternative framing | Free tier (API key) | ✅ All judges |
| `fusion-thirdparty` | claude-haiku-4-5-20251001 (第三方API) | Anthropic diversity, nuanced reasoning | Via 第三方API | ✅ All judges |
| `fusion-budget-ds` | opencode-go/deepseek-v4-flash | Fast budget analysis | $0.14/$0.28 per 1M | ⚠️ Conflicts if Judge = DeepSeek |
| `fusion-budget-mimo` | opencode-go/mimo-v2.5 | Budget broad coverage | $0.14/$0.28 per 1M | ✅ All judges |

### Perspective Assignment (default Judge = DeepSeek V4 Pro):

**Free Tier (default):
- **4-panel free**: fusion-kimi (code architecture) + fusion-qwen (broad/comprehensive) + fusion-gemini (Google diversity, free tier) + fusion-thirdparty (Anthropic diversity)
  *(4 different architectures: Moonshot, Alibaba, Google, Anthropic. Cost: ~$1.35/$5.60 on opencode-go, Gemini free, 第三方API per pricing.)*
- **3-panel free**: fusion-kimi (code architecture) + fusion-qwen (broad/comprehensive) + fusion-gemini (Google diversity, free tier)
- **2-panel free**: fusion-kimi (technical) + fusion-qwen (comprehensive)

**Budget:
- **3-panel budget**: fusion-kimi (technical) + fusion-budget-mimo (broad) + fusion-budget-ds (fast universal)
  *(Note: fusion-budget-ds is DeepSeek V4 Flash, same family as Judge. Judge bias is mild since Flash/Pro are different training variants — acceptable for budget.)*
- **2-panel budget**: fusion-kimi (technical) + fusion-budget-mimo (broad)

**Flagship:
- **3-panel**: fusion-kimi (technical) + fusion-qwen (comprehensive) + fusion-glm (creative/alternative)
- **2-panel**: fusion-kimi (technical) + fusion-qwen (comprehensive)

**Self-Fusion (legitimate fallback — use when no multi-model environment is available):
- **2-panel self (DeepSeek)**: fusion-deepseek (perspective A) + fusion-deepseek (perspective B)
  *(Judge bias intentionally accepted — synthesis step still adds value; DRACO Opus 4.8 case showed +6.7)*
- **2-panel self (Gemini)**: fusion-gemini (perspective A) + fusion-gemini (perspective B)
  *(Optimal for Gemini-native environments when the IDE Main Model is Gemini 3.5 Flash. Self-introspection optimization; benchmark gain not precisely measured for this combo — reference figure is Opus 4.8's +6.7 on DRACO)*

**Note: All DeepSeek-family panels are excluded from default Budget/Flagship because Judge is DeepSeek V4 Pro. To use DeepSeek panels, change Judge to a non-DeepSeek model first. Similarly, for Gemini Self-Fusion, the IDE main model should ideally be set to Gemini 3.5 Flash to orchestrate the subagents.

## Example Usage

User: "Should we use microservices or monolith for our new e-commerce platform?"

AI (with this skill loaded):
1. Recognizes as architecture decision → activate Fusion
2. Designs 3 panel prompts (Judge = DeepSeek V4 Pro, panels avoid DeepSeek):
   - Panel 1 (kimi): "Analyze the TECHNICAL trade-offs: performance, scalability, debugging complexity..."
   - Panel 2 (qwen): "Analyze the ORGANIZATIONAL trade-offs: team structure, deployment, operational cost..."
   - Panel 3 (glm): "Analyze CREATIVE/ALTERNATIVE approaches: hybrid architectures, migration strategies, edge cases..."
3. Dispatches all 3 in parallel via `task` tool
4. Collects responses, produces structured synthesis

## Notes

- Fusion calls have ~2-3x latency vs single model (parallel dispatch mitigates this)
- Not suitable as a coding model replacement — use for research/architecture only
- If only 1 panel succeeds, still produce analysis noting the failure
- Always cite which model said what in the synthesis
