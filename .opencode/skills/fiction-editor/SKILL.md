---
name: fiction-editor
description: Use ONLY for fiction writing, novel chapters, short stories, or narrative content editing. Provides multi-perspective literary critique and produces a final revised draft. Triggers on: 小說, 章節, 故事, fiction, novel, chapter, narrative, prose, 修改文章, 編輯, edit chapter, revise story. Do NOT use for code, technical writing, or factual research.
---

# Fiction Editor — Multi-Model Literary Revision Pipeline

This skill adapts the Fusion multi-model approach for fiction writing. Three specialized literary editors (Plot, Character, Prose) review the text in parallel, then an Editor-in-Chief (Judge) synthesizes their critiques and produces a final revised chapter.

> **適用環境**：本版本（`.opencode/skills/`）使用**不同架構模型**分別擔任三位編輯，需要多模型 token 環境。
> 若你在 Antigravity 等**僅單一底層模型**的 IDE 運行，請改用 `.agents/skills/fiction-editor`（單模型 Self-Fusion 版，靠角色化提示詞區分編輯視角）。

## Core Principle

**一篇小說的修改需要同時關注結構、人物與文筆三個層面。不同模型對不同層面的敏感度不同，平行審稿 + 綜合改稿能產出比單一編輯更好的結果。**

## Workflow

```
使用者提供原文（章節/故事）
        │
        ▼
┌──────────────────────────────────┐
│ Phase 1: Ingestion               │
│ 讀取原文，確認長度與結構          │
└──────────┬───────────────────────┘
           ▼
┌──────────────────────────────────┐
│ Phase 2: Multi-Perspective       │
│ Review (Parallel)                │
│                                  │
│  ┌────────┐  ┌────────┐  ┌────┐ │
│  │ Plot   │  │Character│  │Prose│ │
│  │Editor  │  │Editor   │  │Edit│ │
│  └────┬───┘  └────┬───┘  └─┬──┘ │
│       │           │         │    │
│  ─────┴───────────┴─────────┴── │
│  三份獨立評審報告 (平行產生)      │
└──────────┬───────────────────────┘
           ▼
┌──────────────────────────────────┐
│ Phase 3: Judge Analysis          │
│ Editor-in-Chief 分析三份報告:     │
│  ✅ 共識意見                     │
│  ⚠️ 分歧觀點                     │
│  💡 獨特建議                     │
│  📋 優先修改項目                 │
└──────────┬───────────────────────┘
           ▼
┌──────────────────────────────────┐
│ Phase 4: Final Revision          │
│ 產出:                            │
│  1. 評審摘要 (Review Summary)    │
│  2. 修改後章節全文               │
│     (Final Revised Chapter)      │
└──────────────────────────────────┘
```

## Activation Decision

### Skip Fiction-Editor if:
- Factual/technical editing
- Code or documentation review
- Short proofreading (typos only)
- User asks for single-model edit

### Activate if:
- Full chapter or story revision
- Narrative structure feedback needed
- Character/plot/prose multi-aspect editing
- User submits a chapter/story for critique
- 小說章節修改、故事編輯、長篇稿件審閱

## Phase Details

### Phase 1: Ingestion

- Read the full text from the user's message or referenced file
- Note: author's style, genre, chapter position (if known)
- If text is very long (>8000 words), inform user and handle section by section

### Phase 2: Parallel Review

Launch three panel agents IN PARALLEL. Give each agent:

1. The full original text
2. A perspective-specific instruction (see below)
3. Ask for structured analysis

> **失敗處理**：若某個 panel agent 失敗，在報告中標記 `[FAILED: 編輯角色]`，以剩餘 panel 報告繼續 Phase 3，並在最終輸出開頭說明「X/3 編輯成功」。不放棄已完成的 panel 結果。

#### Panel A: Plot & Structure Editor (`fusion-fiction-plot`)
Focus areas:
- Narrative structure (beginning/middle/end balance)
- Pacing analysis (too fast? too slow? inconsistent?)
- Scene construction (transitions, tension, payoff)
- Plot logic and consistency
- Chapter-level arc (does it serve the overall story?)

#### Panel B: Character & Emotion Editor (`fusion-fiction-character`)
Focus areas:
- Character voice consistency (dialogue, internal thought)
- Emotional depth and reader connection
- Character motivation and behavior logic
- Relationship dynamics
- Dialogue naturalness

#### Panel C: Prose & Style Editor (`fusion-fiction-prose`)
Focus areas:
- Sentence variety and rhythm
- Imagery and sensory detail
- Showing vs telling balance
- Word choice precision and register consistency
- Tone and atmosphere

### Phase 3: Judge Synthesis

After collecting all three reviews, produce a structured analysis:

```
## 📋 評審綜合分析

### ✅ 共識意見
(三位編輯都同意的觀點)

### ⚠️ 分歧意見
| 議題 | 編輯A立場 | 編輯B立場 |
|------|----------|----------|

### 💡 獨特建議
(僅單一編輯提出但有價值的觀點)

### 📊 優先修改項目
按重要性排列的修改項目清單
```

### Phase 4: Final Revision

Produce the final output:

1. **評審摘要** (2-3 paragraphs summarizing key findings)
2. **修改後章節全文** (the complete revised chapter with changes applied)

When generating the revised chapter:
- Incorporate consensus feedback definitely
- For contradictory feedback, use your own literary judgment
- Preserve the author's unique voice and style
- Mark major changes with inline comments `[改: 說明]`
- If the user originally provided the text in Chinese, output the revision in Chinese

> ⚠️ **字數與品質的權衡（重要）**
>
> 根據本專案 Fiction Editor V4 Benchmark 實測（§10 研究報告），字數超標是最常見的「機械扣分」來源——即使文本品質極高（如 fusion-fiction-opus 編輯分 9.00 冠全場，仍因字數超標被拖至 #3）。
>
> - 若使用者**有明確字數限制**（如「約 3000 字」）：Judge 必須在修改後明確核對字數，並在輸出時說明是否符合。不得讓高品質修改因字數失控而被評分懲罰。
> - 若使用者**無字數要求**：以品質為優先，但仍應避免在修改中無謂地大幅擴充原文篇幅。
> - 若優先保留某段精彩內容會超出字數：**明確告知使用者取捨**，由作者決定，不要靜默截斷。

## Available Agents

| Agent | Model | Role | Cost |
|-------|-------|------|------|
| `fusion-fiction-plot` | opencode-go/deepseek-v4-flash | Plot & Structure | $0.14 / $0.28 |
| `fusion-fiction-character` | opencode-go/qwen3.7-plus | Character & Emotion | $0.40 / $1.60 |
| `fusion-fiction-prose` | opencode-go/mimo-v2.5 | Prose & Style | $0.14 / $0.28 |
| Judge (main model) | opencode-go/deepseek-v4-pro | Editor-in-Chief | — |

> Note: Update `opencode.jsonc` to register `fusion-fiction-*` agents before use. See configuration section.

## Example

User pastes a chapter and asks:
> 「幫我 review 這篇小說第四章，給修改建議後產出最終版」

AI:
1. Ingests the chapter text
2. Dispatches 3 agents in parallel:
   - `fusion-fiction-plot`: structural analysis
   - `fusion-fiction-character`: character/dialogue analysis
   - `fusion-fiction-prose`: prose/stylistic analysis
3. Synthesizes reviews
4. Outputs: Review Summary + Full Revised Chapter

## Design Notes

- Judge (Editor-in-Chief) uses DeepSeek V4 Pro to avoid panel overlap
- All three panel models are different architectures from the Judge (no self-bias)
- If only 2 of 3 panels succeed, still produce revision with note
- For very long chapters, consider section-by-section processing
