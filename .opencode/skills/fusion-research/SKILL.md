---
name: fusion-research
description: Use ONLY when the user asks complex research, architecture design, multi-faceted analysis, or strategic decision questions. Implements OpenRouter Fusion-style multi-model collaborative reasoning. Triggers on keywords: research, analyze deeply, compare models, architecture decision, multi-perspective, pros and cons, evaluate options, 分析, 研究, 比較, 評估. Do NOT use for simple coding, file operations, or single-answer questions.
---

# Fusion Research - Multi-Model Collaborative Analysis

This skill implements a multi-model "Fusion" pipeline inspired by OpenRouter Fusion, adapted for opencode's agent system. When a complex question is detected, dispatch it to multiple panel agents (each with a different model architecture) in parallel, then act as Judge to synthesize the results.

## Core Principle

**不同架構的模型有不同的推理風格與擅長領域，同時發問、平行收集、綜合裁判，能有效涵蓋彼此的盲點。**

Reference research data (OpenRouter Fusion DRACO benchmark):
- Solo DeepSeek V4 Pro: 60.3%
- Self-Fusion (same model ×2): 65.5% (+5.2)
- Multi-model Fusion (3 diverse models): 64.7% ~ 69.0%
- Best Solo (Claude Fable 5): 65.3%

### ⚠️ Critical Rule: Judge Bias Avoidance

**Judge 必須與所有 Panel 使用不同架構的模型**，否則會產生系統性偏差（模型傾向認同自己的輸出，即使提示詞不同）。

OpenRouter Fusion 研究報告指出 Judge 偏差會造成 10~25 分的絕對分數差異。

- 當 Judge 為 DeepSeek V4 Pro 時 → Panel 不得包含 DeepSeek 系列模型
- 當 Judge 為 GLM-5.2 時 → Panel 不得包含 GLM 系列模型
- **Self-Fusion** 是唯一例外（同模型做 Judge + Panel，+5.2 分）。這是因為合成步驟本身新增價值，但仍需意識到存在偏差

## Activation Decision

ALWAYS evaluate the user's question against this triage checklist BEFORE answering:

### Skip Fusion (answer directly) if:
- Simple coding task (write a function, fix a bug)
- File read/write operations
- Single-fact lookup
- Short, unambiguous question
- User explicitly requests a single model answer

### Activate Fusion if ANY of these are true:
- Research/deep analysis question
- Architecture or design decision
- Multi-faceted comparison or evaluation
- Strategic planning with trade-offs
- Question where diverse perspectives add value
- User asks for "comprehensive analysis" or similar
- 使用者用中文問複雜問題

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

### Phase 3: Judge Synthesis

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
| `fusion-qwen` | opencode-go/qwen3.7-max | Comprehensive analysis, broad context | $2.50/$7.50 per 1M | ✅ All judges |
| `fusion-glm` | opencode-go/glm-5.2 | Creative thinking, alternative angles | $1.40/$4.40 per 1M | ✅ All judges |
| `fusion-budget-ds` | opencode-go/deepseek-v4-flash | Fast budget analysis | $0.14/$0.28 per 1M | ⚠️ Conflicts if Judge = DeepSeek |
| `fusion-budget-mimo` | opencode-go/mimo-v2.5 | Budget broad coverage | $0.14/$0.28 per 1M | ✅ All judges |

### Perspective Assignment (default Judge = DeepSeek V4 Pro):

**Budget (default):
- **3-panel budget**: fusion-kimi (technical) + fusion-budget-mimo (broad) + fusion-budget-ds (fast universal)
  *(Note: fusion-budget-ds is DeepSeek V4 Flash, same family as Judge. Judge bias is mild since Flash/Pro are different training variants — acceptable for budget.)*
- **2-panel budget**: fusion-kimi (technical) + fusion-budget-mimo (broad)

**Flagship:
- **3-panel**: fusion-kimi (technical) + fusion-qwen (comprehensive) + fusion-glm (creative/alternative)
- **2-panel**: fusion-kimi (technical) + fusion-qwen (comprehensive)

**Self-Fusion (special case):
- **2-panel self**: fusion-deepseek (perspective A) + fusion-deepseek (perspective B)
  *(Judge bias intentionally accepted — synthesis steps still provide +5.2 gain)*

**Note: All DeepSeek-family panels are excluded from default Budget/Flagship because Judge is DeepSeek V4 Pro. To use DeepSeek panels, change Judge to a non-DeepSeek model first.

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
