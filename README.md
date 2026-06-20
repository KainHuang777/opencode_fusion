# OpenRouter Fusion — opencode 多模型協同系統

> 基於 [OpenRouter Fusion 研究報告](./openrouter_fusion_research.md) 設計的 opencode Skill + Agent 實作方案

## 概念

將同一任務派送給多個不同架構的模型，由 Judge 分析共識與差異後產出綜合結果。三個臭皮匠勝過一個諸葛亮。

## 適用場景

| 場景 | 相對單一模型增益 |
|------|----------------|
| 研究分析、架構決策 | +5~10 分 |
| 小說章節編輯修訂 | +8~15 分（三位編輯視角互補） |
| 複雜策略評估 | +3~7 分 |
| 簡單問答 | 無增益，直接回答 |

## 快速開始

```bash
# 1. 確認 opencode-go provider 可用
opencode models
# 若無 → /connect → 選 OpenCode Go → 輸入 API key

# 2. 接入 Google Gemini（free tier，可選）
# 前往 https://aistudio.google.com/apikey 取得 API Key
# opencode 內 → /connect → 選 Google → 貼上 API Key

# 3. 接入 Claude Haiku 4.5（第三方API，可選）
# 取得 API Key，設定 endpoint（OpenAI-compatible）
# opencode 內 → /connect → 選對應 provider → 貼上 API Key + 端點

# 4. 重啟 opencode（配置啟動時載入）
opencode
```

## 檔案結構

```
E:\WORK\Fusion\
├── opencode.jsonc                    # 專案配置（agent + skill 註冊）
├── openrouter_fusion_research.md    # 原始研究報告
├── README.md                        # 本文件
└── .opencode/
    ├── agents/
    │   ├── fusion-deepseek.md       # [Research] 技術深度 Panel (V4 Pro)
    │   ├── fusion-kimi.md           # [Research] 架構分析 Panel (K2.7 Code)
│   ├── fusion-qwen.md           # [Research] 綜合分析 Panel (Qwen3.7 Plus)
│   ├── fusion-glm.md            # [Research] 創意思考 Panel (GLM-5.2)
│   ├── fusion-gemini.md         # [Research] Google 多樣性 Panel (Gemini 3.5 Flash, free tier)
│   ├── fusion-skyunion.md       # [Research] Anthropic 多樣性 Panel (Claude Haiku 4.5, 第三方API)
    │   ├── fusion-budget-ds.md      # [Research] 平價 Panel (V4 Flash)
    │   ├── fusion-budget-mimo.md    # [Research] 平價 Panel (MiMo V2.5)
    │   ├── fusion-fiction-plot.md   # [Fiction] 結構編輯 (V4 Flash)
    │   ├── fusion-fiction-character.md  # [Fiction] 人物編輯 (Qwen3.7 Plus)
    │   └── fusion-fiction-prose.md  # [Fiction] 文筆編輯 (MiMo V2.5)
    └── skills/
        ├── fusion-research/
        │   └── SKILL.md             # 研究分析工作流
        └── fiction-editor/
            └── SKILL.md             # 小說編輯工作流
```

## 工作流一：Fusion Research（研究分析）

多模型平行分析 + Judge 綜合，適用於架構決策、技術調研、策略評估。

```
使用者提問
    │
    ▼
判斷觸發 Fusion？
  ├─ 簡單問題 → 直接回答
  └─ 複雜問題 →
       Phase 1: 為每個 Panel 設計不同視角提示詞
       Phase 2: 平行派遣 → task(fusion-*) × 3
       Phase 3: Judge 分析共識/分歧/盲點
       Phase 4: 輸出結構化報告
```

### 可用 Panel

| 名稱 | 模型 | 強項 | 成本 | Judge 相容 |
|------|------|------|------|-----------|
| `fusion-deepseek` | V4 Pro | 技術深度、邏輯推理 | $1.74/$3.48 | ⚠️ 衝突 |
| `fusion-kimi` | K2.7 Code | 程式架構、實作思維 | $0.95/$4.00 | ✅ |
| `fusion-qwen` | Qwen3.7 Plus | 綜合分析、廣度覆蓋 | $0.40/$1.60 | ✅ |
| `fusion-glm` | GLM-5.2 | 創意思考、反向論證 | $1.40/$4.40 | ✅ |
| `fusion-gemini` | Gemini 3.5 Flash | Google 架構多樣性 | Free tier | ✅ |
| `fusion-skyunion` | Claude Haiku 4.5 | Anthropic 架構、細膩推理 | 第三方API | ✅ |
| `fusion-budget-ds` | V4 Flash | 快速技術分析 | $0.14/$0.28 | ⚠️ 衝突 |
| `fusion-budget-mimo` | MiMo V2.5 | 廣度快速覆蓋 | $0.14/$0.28 | ✅ |

> Judge 預設為 DeepSeek V4 Pro。任何使用同家族模型的 Panel 會產生輕微裁判偏差。

## 工作流二：Fiction Editor（小說編輯）

三位文學編輯平行審稿 + Editor-in-Chief 綜合改稿。專用於小說章節、短篇故事修改。

```
使用者提供原文
    │
    ▼
  Phase 1: 三位編輯平行評審
    Plot Editor (結構/節奏/場景邏輯)
    Character Editor (人物/對白/情感)
    Prose Editor (文筆/意象/句式)
    │
    ▼
  Phase 2: Editor-in-Chief 綜合分析
    共識意見 / 分歧觀點 / 優先修改項目
    │
    ▼
  Phase 3: 產出最終版本
    評審摘要 + 修改後全文
```

### 三種編輯視角（極省版）

| 編輯 | 模型 | 審查重點 | 成本 |
|------|------|---------|------|
| `fusion-fiction-plot` | DeepSeek V4 Flash | 情節結構、節奏、場景邏輯 | $0.14/$0.28 |
| `fusion-fiction-character` | Qwen3.7 Plus | 人物一致性、情感深度、對白 | $0.40/$1.60 |
| `fusion-fiction-prose` | MiMo V2.5 | 文筆風格、意象、句式節奏 | $0.14/$0.28 |
| Judge | DeepSeek V4 Pro | 總編輯，綜合改稿 | — |

> 每輪成本約 **$0.68 in / ~$2.16 out**，靠 Qwen3.7 Plus 撐中文人物品質，比旗艦版便宜 7 倍。

### Benchmark 實測成績

> 以下數據摘自 V3 vs V4 小說模型評測（2026-06-20），22 個模型同一批正文，分別用兩套評審體系打分。結果顯示：**Fusion 策略不僅評分更高，而且成本更低。**

**Top 5 排名：**

| 排名 | 模型 | V4 得分 | 類型 | 相對成本 |
|:---:|------|:-------:|------|:-------:|
| #1 | kimi-k2.5 | 8.63 | 單模型 | 高 |
| #2 | gtp5.5 | 8.60 | 單模型 | 極高 |
| **#3** | **fusion-fiction-opus** | **8.50** | **Fusion（3編輯 + 總編）** | **中** |
| #4 | glm-5 | 8.50 | 單模型 | 中 |
| **#5** | **fusion-budget** | **8.44** | **Fusion 平價組** | **極低** |

> fusion-budget 的正文由 2 個 V4 Flash panel 平行起草 + V4 Pro Judge 綜合而成（非使用他人正文）；fusion-fiction-opus 則是以 claude-4.6-opus 的原文為底本，經三位編輯 + 總編改稿。兩者路徑不同，但都驗證了同一件事：**組合策略以遠低於單一旗艦模型的成本，產出可比肩頭部的品質**。

**V3 → V4 分數提升（拆掉「跨維度盲評」這堵牆）：**

| 維度 | V3 均分 | V4 均分 | 提升 | 對應專業編輯 |
|------|:-------:|:-------:|:----:|--------------|
| 節奏感 | 8.15 | 8.85 | **+0.70** | Plot Editor |
| 角色塑造 | 7.75 | 8.28 | **+0.52** | Character Editor |
| 開篇吸引力 | 8.47 | 8.95 | **+0.47** | Plot Editor |
| 文筆 | 8.50 | 8.74 | +0.24 | Prose Editor |
| 對話品質 | 7.89 | 8.03 | +0.13 | Character Editor |

> V3 的通用評委經常出現跨維度盲評（擅長節奏感的評委亂打分數幽默感）。V4 將每位編輯鎖死在專長維度上，讓最懂該維度的人打該維度的分——噪音被系統性壓縮。

**成本結構：「啞鈴型架構」的 Token 經濟學**

```
          廣度層（便宜專才）               深度層（少量綜合）
  ┌────────────────────────┐     ┌──────────────────┐
  │  Plot Editor           │     │                  │
  │  (DS V4 Flash)         │───▶│                  │
  │  只評節奏/開篇          │     │  Editor-in-Chief │
  │  $0.14/$0.28 per 1M   │     │  (DS V4 Pro)     │
  ├────────────────────────┤     │  綜合三份精煉報告  │
  │  Character Editor      │     │  $1.74/$3.48     │
  │  (Qwen3.7+)            │───▶│  per 1M          │
  │  只評角色/對話/幽默      │     │                  │
  │  $0.40/$1.60 per 1M   │     └──────────────────┘
  ├────────────────────────┤
  │  Prose Editor          │
  │  (MiMo V2.5)           │───▶
  │  只評文筆/年代感         │
  │  $0.14/$0.28 per 1M   │
  └────────────────────────┘
```

關鍵差異：每位編輯的輸出被限定在自己專長的 2-3 個維度；總編不需通讀全文，只需消化三份精煉報告。**全鏈條 token 消耗低於「4 人完整評審」，品質卻更高。**

**量化對比：Fusion vs 單一引擎**

| 方法 | 模型組合 | 近似成本 | V4 成績 |
|------|---------|:--------:|:------:|
| 單模型 | kimi-k2.5 獨寫 | 高 | 8.63 (#1) |
| 單模型 | deepseek-v4-pro 獨寫 | 中 | 8.15 (#11) |
| Fusion | fusion-budget（2 Flash panels + Pro Judge） | **極低** | 8.44 (#5) |
| 單模型 | claude-4.6-opus 獨寫 | 極高 | 7.95 (#16) |

> fusion-budget 以不到單模型頭部十分之一的推理成本擠進前五。**AI 文學創作正在從「比誰的引擎排量大」轉向「比誰的傳動系統設計得好」**。

### 使用範例

```
請幫我 review 這篇小說第三章，給修改建議後產出最終版

[貼上章節內容]
```

## 方案比較

| 方案 | Judge | Panel 1 | Panel 2 | Panel 3 | Panel 4 | 每輪成本 | 適用 |
|------|-------|---------|---------|---------|---------|---------|------|
| **A Free Tier** ⭐ | V4 Pro | K2.7 Code | Qwen3.7 Plus | Gemini 3.5 Flash | Claude Haiku 4.5 | ~$1.35/$5.60 + free + 第三方API | 日常使用（預設） |
| **B 旗艦** | V4 Pro | K2.7 Code | Qwen3.7 Plus | GLM-5.2 | — | ~$2.75/$11.00 | 最高品質研究 |
| **C 全 Go 平價** | V4 Pro | K2.7 Code | MiMo V2.5 | V4 Flash | — | ~$1.23/$4.56 | 無 Gemini 時 |
| **D Self-Fusion** | V4 Pro | V4 Pro(A) | V4 Pro(B) | — | — | ~$1.74/$3.48 | 快速驗證 |

> A 方案為預設 free tier 組合：四種不同架構（Moonshot, Alibaba, Google, Anthropic）。Gemini 3.5 Flash 經 Google AI Studio free API key 接入，Claude Haiku 4.5 經第三方API接入。

## 切換方案

編輯 `opencode.jsonc` 中 panel agent 的 `model` 欄位，重啟生效。

```jsonc
"fusion-kimi": {
  "model": "opencode-go/kimi-k2.7-code"
},
"fusion-qwen": {
  "model": "opencode-go/qwen3.7-plus"
},
"fusion-skyunion": {
  "model": "第三方-provider/claude-haiku-4-5-20251001"  // 依你的第三方API provider 設定
}
```

## 注意事項

### Judge 偏差
- Judge（DeepSeek V4 Pro）不得與 Panel 使用同架構模型
- 同型號偏差 10~25 分；V4 Flash/V4 Pro 為不同訓練變體，偏差較輕微
- Self-Fusion（方案 C）故意接受偏差換取速度

### 其他
- Fusion 延遲約為普通請求 2-3 倍（平行派遣可緩解）
- 部分 panel 失敗時仍產出分析，標註失敗模型
- 所有合成結論標註各觀點來源

## 效能參考（DRACO Benchmark）

| 組合 | 分數 |
|------|------|
| Fusion 旗艦 (Fable 5 + GPT-5.5) | **69.0%** |
| Fusion 三模型 (Opus 4.8 + GPT-5.5 + Gemini 3.1 Pro) | **68.3%** |
| Fusion 雙模型 (Opus 4.8 + GPT-5.5) | **67.6%** |
| 單一最強 Claude Fable 5 | 65.3% |
| Fusion 平價 (Gemini Flash + Kimi + DeepSeek) | **64.7%** |
| 單一基準 DeepSeek V4 Pro | 60.3% |

## 參考資料

- [OpenRouter Fusion Blog](https://openrouter.ai/blog/announcements/fusion-beats-frontier/)
- [opencode-go 文檔](https://opencode.ai/docs/go/)
- [opencode Agent 文檔](https://opencode.ai/docs/agents/)
- [opencode Skill 文檔](https://opencode.ai/docs/skills/)
- [DRACO Benchmark (arXiv)](https://arxiv.org/abs/2602.11685)
