---
name: fiction-editor
description: (Self-Fusion / 單模型版) Use ONLY for fiction writing, novel chapters, short stories, or narrative content editing in single-model IDE environments (e.g. Antigravity). Provides multi-perspective literary critique via role-based subagents on the same underlying model, then produces a final revised draft. Triggers on: 小說, 章節, 故事, fiction, novel, chapter, narrative, prose, 修改文章, 編輯, edit chapter, revise story. Do NOT use for code, technical writing, or factual research.
---

# Fiction Editor - Antigravity 2.0 Multi-Perspective Literary Revision Pipeline

本技能專為 **Antigravity 2.0** 系統設計。在無法跨越底層大模型的限制下，透過定義三個不同視角的虛擬編輯子代理（編劇、塑造大師、文筆大師），平行審查小說原文，再由主對話代理（總編輯 Judge）綜合他們的意見，產出高水準的最終修改草稿。

> **適用環境**：本版本（`.agents/skills/`）為**單模型 Self-Fusion** 設計，三位編輯均基於同一底層模型，僅靠角色化提示詞區分視角，適用於 Antigravity 等無法切換模型的 IDE。
> 若你**有多個模型可用**（opencode-go / Google / 第三方 API），請改用 `.opencode/skills/fiction-editor`（真·多模型版，由不同架構模型分別擔任編輯，盲點互補效果更佳）。
> 效能優先序：**不同架構模型 > 同模型 ×2（本版）> 單一模型直答**。

## 核心原則

一篇優秀的小說修改需要同時關注「情節結構」、「人物塑造」與「文筆修辭」三個維度。雖然在 Antigravity 中所有編輯均基於同一個底層大模型，但藉由在 `system_prompt` 中賦予極為具體的寫作偏好與評審視角，子代理能提供比單一回答深邃數倍的批註。

---

## 啟動決策 (Activation Decision)

### ❌ 跳過 Fiction-Editor (直接回答)
- 程式碼審查、編譯出錯排查或技術問題。
- 非敘事性的技術文件、行銷文案或事實性學術寫作。
- 單純的錯字/語法微調（不需要結構性修改）。
- 使用者要求單一模型快速潤飾。

###   啟用 Fiction-Editor (啟動本技能)
- 完整的小說章節修改與評審。
- 需要針對情節結構與節奏給予改進意見。
- 需要深入刻畫角色對話、情感張力或人物性格。
- 使用者上傳一整段故事，並要求「編輯」、「改寫」或「優化文筆」。

---

## 工作流階段 (Workflow Phases)

### Phase 1: 原文讀取與角色定義
讀取使用者輸入的小說原文。在 Antigravity 2.0 中，主代理調用 `define_subagent` 定義以下三個專業編輯子代理：

1.  **Plot & Structure Editor (`plot-editor`)**：
    - **定位**：情節與節奏把關者。
    - **專攻**：起承轉合結構、場景切換、懸念與衝突設計、敘事節奏（太快/太慢/拖沓）、邏輯合理性。
2.  **Character & Emotion Editor (`character-editor`)**：
    - **定位**：人物性格與情感刻畫大師。
    - **專攻**：角色對話語氣是否符合人設、內心活動是否飽滿、情感變化是否合理自然、角色關係動態。
3.  **Prose & Style Editor (`prose-editor`)**：
    - **定位**：文筆與修辭修潤者。
    - **專攻**：用詞精準度、句式多樣性、氛圍感營造、冗詞贅句修剪、文風流暢度。

---

### Phase 2: 平行評審派發 (Parallel Review)
主代理調用 `invoke_subagent` 平行啟動上述三個編輯子代理，傳遞小說原文並要求產出結構化的「評審報告」。

每個子代理須產出：
1. **維度評分**：該維度（如文筆）的 1-10 分評估與理由。
2. **具體批註**：具體到某行或某段的修改意見（指明優缺點）。
3. **具體改寫範例**：針對有瑕疵的段落給出改寫示範。

> **失敗處理**：若某個子代理失敗，在報告中標記 `[FAILED: 編輯角色]`，以剩餘報告繼續 Phase 3，並在最終輸出開頭說明「X/3 編輯成功」。已完成的子代理結果不得丟棄。

---

### Phase 3: 總編輯 (Judge) 綜合改稿 (Final Synthesis)
主代理收集三份報告後，進行對比、歸納共識，並產出綜合性修改成果：

```markdown
## ✍️ Fiction Editor 綜合修訂報告 (Antigravity Self-Fusion)

### 📋 編輯部評審摘要
| 編輯角色 | 關注維度 | 核心診斷 | 評分 (1-10) |
|------|------|------|:---:|
| plot-editor | 情節與結構 | ... | /10 |
| character-editor | 人物與情感 | ... | /10 |
| prose-editor | 文筆與修辭 | ... | /10 |

### ✅ 共識修改建議
三位編輯一致認同需要調整的段落或方向：
- ...

### ⚠️ 分歧修改建議
例如情節編輯認為某段對話需要刪減以加快節奏，而人物編輯認為該對話是展現人設的關鍵。總編輯在此需要進行裁決：
- ...

### 💡 總編輯最終修改版 (Final Revised Chapter)
以下是結合了所有編輯意見後，修訂完成的全新章節內容：

[ 在此輸出修改後的完整小說正文 ]
```

> ⚠️ **字數與品質的權衡（重要）**
>
> - 若使用者**有明確字數限制**（如「約 3000 字」）：總編輯須在修改後核對字數，並明確說明是否符合。
> - 若優先保留某段精彩內容會超出字數：**明確告知使用者取捨**，由作者決定，不要靜默截斷。
> - 若使用者**無字數要求**：以品質為優先，但避免無謂地大幅擴充原文篇幅。
>
> 根據 Fiction Editor V4 Benchmark 實測，字數超標是最常見的「機械扣分」來源，高品質修改可能因此被埋沒——總編輯有責任主動管理這個風險。
