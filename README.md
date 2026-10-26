# harness-research

一個開源 agent harness 的前期研究：想打入開源市場，面向技術不那麼高的 vibe coder，同時讓專業開發者也覺得好用。

**目前狀態：研究階段，還沒有任何程式碼。** 產品名稱、定位與 MVP 範圍都還沒定案。

## 目前的研究結論（摘要）

**市場：** 通用型 coding agent harness 已經很擠，功能面（模型中立、MCP、`AGENTS.md`、plan 模式、context 壓縮）幾乎同質化，做了不會加分。

**缺口：** 在調查過的 8 個 coding 側產品與 8 個助理側產品裡，沒有任何一個同時做到「預設安全」、「做錯能還原」、「成本可控」且對新手友善。

- 預設值普遍偏寬鬆（多個產品預設自動核准或不設沙箱）。
- 「撤銷已發生的事」幾乎是空白；能看到發生了什麼，比能還原做得好得多。
- 免 API key 的入口幾乎都導向各家的商業 gateway。
- 常駐助理型產品的攻擊面遠大於 coding harness（例如 OpenClaw 的 GitHub advisory 約 730 筆（由分頁數推算），其中 14 筆 Critical，集中在權限範圍與核准的授權邏輯）。

**目前傾向的做法（尚未定案）：** 先以 coding harness 的範圍起步（本機、單次 session、不做訊息 gateway 與常駐排程），但架構上從一開始就採用常駐助理也需要的安全模型：確定性權限、憑證分離、網路出口預設鎖、可還原的 snapshot、產出物放在使用者自己的 repo。

## 研究內容

| 檔案 | 內容 |
|---|---|
| [`research/comparison.md`](research/comparison.md) | **Coding 側比較**：8 個開源 harness 的跨工具比較、共通點、缺口、值得抄的設計與要避開的坑 |
| [`research/assistants/comparison.md`](research/assistants/comparison.md) | **助理側比較**：8 個個人助理型產品的比較、OpenClaw 安全事件史、從中推導的最低安全門檻、對定位決策的含義 |

個別報告：

- Coding 側（`research/`）：[opencode](research/opencode.md)、[cline](research/cline.md)、[kilo-code](research/kilo-code.md)、[aider](research/aider.md)、[codex-cli](research/codex-cli.md)、[goose](research/goose.md)、[openhands](research/openhands.md)、[hermes-agent](research/hermes-agent.md)
- 助理側（`research/assistants/`）：[openclaw](research/assistants/openclaw.md)、[nanoclaw](research/assistants/nanoclaw.md)、[zeroclaw](research/assistants/zeroclaw.md)、[vellum](research/assistants/vellum.md)、[claude-cowork](research/assistants/claude-cowork.md)、[chatgpt-agent](research/assistants/chatgpt-agent.md)、[manus](research/assistants/manus.md)、[zo-computer](research/assistants/zo-computer.md)

## 研究的限制（引用前請先看）

- 查證日期為 2026-10-25 至 10-26。這個領域變動極快，star 數、版本、價格、功能很快會過時。
- 多數官方網站在查證環境無法連線，報告改讀 repo 內的 markdown 與原始碼；Manus、Zo、ChatGPT 的報告只靠搜尋摘要，其中 ChatGPT 的可信度最低。
- **沒有人實際安裝過任何產品**，所有首跑體驗、核准提示頻率、輸出品質都是文件推論。
- Issue 與 advisory 多數只讀到標題或摘要；暴露實例數、惡意 skill 數等為二手且互相矛盾的數字，請勿引用。
- 每份報告都分開標示「已驗證」、「推論」、「二手」與「未驗證」；沒有驗證到的內容寫的是 unverified，沒有用記憶補齊。

## 接下來

1. 挑 2 到 3 個產品實際安裝，用同一個新手任務錄下首跑體驗與核准提示頻率。
2. 根據缺口規劃 MVP 範圍與威脅模型。
3. 決定產品名稱與定位（coding harness，或個人軟體助理）。
