---
name: fusion-research
description: (Self-Fusion / 單模型版) Use ONLY when the user asks complex research, architecture design, multi-faceted analysis, or strategic decision questions in single-model IDE environments (e.g. Antigravity). Implements Antigravity 2.0 Self-Fusion style multi-perspective collaborative reasoning on the same underlying model. Triggers on keywords: research, analyze deeply, compare models, architecture decision, multi-perspective, pros and cons, evaluate options, 分析, 研究, 比較, 評估. Do NOT use for simple coding, file operations, or single-answer questions.
---

# Fusion Research - Antigravity 2.0 Self-Fusion Pipeline

本技能專為 **Antigravity 2.0** 系統設計。在無法跨越底層大模型的限制下，透過將同一任務分發給多個「不同提示詞與角色定位」的子任務代理 (Subagents) 平行執行，再由主對話代理擔任 Judge 進行共識與分歧分析，以重建 **Self-Fusion** 的雙重驗證優化效果。

> **關於 Self-Fusion 的定位**：同模型 ×2 + 同模型 Judge 是 OpenRouter 原始研究中正式存在的合法組合（DRACO 上 Opus 4.8 達 +6.7 分，58.8%→65.5%），並非禁忌。優先順序為「**不同架構模型 > 同模型 ×2 > 單一模型**」。本技能用於 Antigravity 這類底層僅有單一模型、無法跨模型切換的環境；此時 Self-Fusion 仍優於單模型直接回答，惟須注意本環境下的實際增益並未經明確 Benchmark 量化。若你的環境具備多模型 token，請優先改用 `.opencode/skills/fusion-research`（真·多模型版本）。

## 核心原則

- **多視角分工**：即使是同一個底層模型，透過不同的系統提示詞（System Prompts），可以激活模型內不同的參數與知識區塊，產出具備多樣性的回應。
- **隨機幻覺緩解**：透過平行兩至三次的獨立調用，可以有效交叉驗證，過濾隨機產生的代碼 Bug 或幻覺。
- **主代理統合 (Meta-Reasoning)**：主對話代理收集子代理報告後，進行對比、歸納共識與挑出分歧，可強迫模型進行二次推理，從而提升輸出嚴謹度。

---

## 啟動決策 (Activation Decision)

在回答使用者問題前，先快速評估**複雜度 × 不確定性**：

### ❌ 跳過 Fusion（直接回答）
- 簡單的編碼或除錯任務（如：寫一個函數、修改語法錯誤）。
- 單純的文件讀寫或操作命令。
- 單一事實查詢，或問題有客觀正確答案（如：「A 函數效能比 B 快？」）。
- 使用者明確要求直接回答。

###   啟用 Fusion（同時滿足：多面向 + 高不確定性）
- 複雜的技術調研或架構決策，且不同視角會得出不同建議。
- 涉及多個技術方案的主觀權衡（A vs B，無明顯客觀勝者）。
- 具有複雜業務邏輯的系統設計。
- 使用者要求「深入分析」、「多角度評估」或提問極為複雜。

> **防止誤觸發**：「分析」、「比較」等詞單獨出現不足以啟動。
> 若問題有客觀答案，直接回答更高效，不需 Self-Fusion 增加 token 消耗。

---

## 工作流階段 (Workflow Phases)

### Phase 1: 角色與提示詞設計
分析使用者的核心問題，設計 2-3 個專注於不同維度的子代理角色與 Prompts：
- **子代理 A (技術深度與代碼細節)**：專注於邏輯正確性、算法效率、代碼邊界條件與潛在的並發/安全隱患。
- **子代理 B (架構設計與可擴展性)**：專注於整體模組解耦、設計模式應用、代碼可維護性與長期架構演進。
- **子代理 C (創意思考與替代方案)**：專注於質疑前提假設、提出非主流的優化思路、逆向思維與極端邊界測試。

---

### Phase 2: 子代理定義與平行派發 (Subagent Dispatch)
在 Antigravity 2.0 中，使用 `define_subagent` 定義這三個角色，並以 `invoke_subagent` 平行啟動：

1. **定義子代理**（範例）：
   - 調用 `define_subagent` 定義 `tech-detail-panel`，賦予其專門的 `system_prompt`（著重於代碼深度與正確性）。
   - 調用 `define_subagent` 定義 `architecture-panel`，賦予其專門的 `system_prompt`（著重於架構與耦合度）。
   - 可選：定義 `alternative-panel`，賦予其反向論證的 `system_prompt`。

2. **平行調用子代理**：
   - 調用 `invoke_subagent` 將使用者的核心問題與視角 Prompt 派發給上述定義的子代理，同時在背景運行。

---

### Phase 2.5: 失敗處理

- **部分子代理失敗**：在報告中標記 `[FAILED: 子代理名]`，以剩餘報告繼續 Judge 合成，並在開頭說明「X/N 視角成功」。
- **所有子代理失敗**：告知使用者，建議簡化問題或直接回答。
- **本階段不應放棄已產出的任何子代理回應**。

---

### Phase 3: 主代理 Judge 綜合裁判 (Synthesis)

> ⚠️ **Self-Fusion Judge 偏差緩解（強制執行）**
>
> 由於 Judge 與 Panel 使用同一底層模型，存在固有偏差（傾向認同自己的輸出）。Judge 在合成時**必須**：
> 1. **強制批判**：至少明確指出各 panel 中 **≥2 個不合理、過度工程化或論據薄弱**之處（不允許全盤認同 panel 輸出）。
> 2. **Hard Checkpoints**（視問題類型）：
>    - 代碼問題：程式碼是否能通過編譯？有無並發死鎖或 SQL 注入風險？
>    - 架構問題：有無引入不必要的複雜度？有無違反被提到的設計原則？
>    - 研究問題：有無引用不存在的數據或過時資訊？結論有無直接因果依據？
> 3. **結論要與 panel 有所差異**：最終裁決須整合各角度，而非直接複製任一 panel 的觀點。

當所有子代理完成並返回報告後，主代理（您）需進行整合，並輸出結構化報告：

```markdown
## 🔬 Fusion 綜合分析報告 (Antigravity Self-Fusion)

### 📊 參與 Panel 視角
| 代理名稱 | 視角定位 | 核心摘要 |
|------|------|------|
| tech-detail-panel | 技術深度與代碼細節 | ... |
| architecture-panel | 架構設計與可維護性 | ... |
| alternative-panel | 逆向思維與替代方案 | ... |

###  共識 (Consensus)
所有或大部分 Panel 均同意的技術觀點或解決路徑：
- ...

### ⚠️ 分歧與衝突 (Contradictions)
不同視角 Panel 之間存在矛盾的建議或權衡點：
- ...

### 💡 獨特見解與盲點 (Unique Insights & Blind Spots)
在本次雙重驗證中被挖掘出的關鍵隱患或非常規優化方案：
- ...

### 📋 最終裁決與最佳推薦方案
主代理結合上述所有分析，給出的最終整合代碼或解決方案：
```

---

## 注意事項與成本控制
- **遞迴限制**：禁止 Panel 子代理再次調用 Fusion 技能（`fusion-depth` 限制為 1），防止 API Token 消耗爆炸。
- **裁判偏好 (Judge Bias)**：需意識到由於 Judge 與 Panel 同模型，主代理可能會對子代理產生的錯誤觀點缺乏完全獨立的客觀性，應在 Synthesis 階段保持高標準的審查和驗證。
