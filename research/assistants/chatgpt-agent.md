# OpenAI ChatGPT agent / ChatGPT Work 研究筆記

> **重要方法聲明（資料可信度較低）**：本環境無法連到 OpenAI 的主要網站——WebFetch 對 help.openai.com、openai.com、cdn.openai.com、deploymentsafety.openai.com 全部回傳 `getaddrinfo ENOTFOUND`，curl 經 proxy 回 `CONNECT tunnel failed, 403`。因此本文所有 OpenAI 資訊都來自 **WebSearch 的結果摘要**（摘要內容源自 OpenAI 官方頁面，但經搜尋工具改寫，非逐字原文，且我們沒有開啟過任何一頁原文）。凡引號內英文，皆為搜尋摘要中標為引用的字句，仍可能與原頁有出入。與 Cowork 檔案相比，本檔精確度明顯較低，請視為「待驗證線索」。無法實際使用產品，皆為文件層級。
> 檢查日期：2026-10-26。

## 1. 基本資料

**命名與產品演變（最重要的釐清）**：
- **Operator**（2025-01）：獨立瀏覽器 agent；其核心功能後被整合進 ChatGPT agent，operator.chatgpt.com 預定停用（help 文章 release notes 摘要）。
- **ChatGPT agent**：2025-07-17 推出，先給 Pro、Plus、Team；2025-08-08 加入 Enterprise 與 Edu（release notes 摘要）。
- **ChatGPT Work**：release notes 摘要載明「Introducing ChatGPT Work」於 **2026-07-09** 推出，web 與 mobile 對付費方案（排除 Free、Go）先給 Pro、Pro Lite、Enterprise、Edu，Plus 與 Business 隨後；Enterprise/Edu 有兩週預覽、預設關閉。新的 ChatGPT desktop app（macOS、Windows）整合 Chat、Work、Codex。另一方面，**help 文章 "ChatGPT agent" 的概述段落稱 agent 已不再提供，並導向 ChatGPT Work**；但同頁可用性段落仍列 agent mode 與每月訊息限制，頁面自我矛盾（搜尋摘要所見，未驗證）。社群貼文稱 agent mode 已被移除（使用者回報，非官方）。結論：**目前主打且持續更新的是 ChatGPT Work；ChatGPT agent/Operator 為前身，應視為已被取代或正在被取代**。
- 其他：Wikipedia 稱 agent 於 2026-08 初移除（二手，未驗證）。

**方案與價格**（搜尋摘要，各來源有出入，均未驗證）：Plus $20/月；Pro 有 $100、$200、$500 檔（有說明稱 $200 新訂暫停，可能過期）；Business Standard seat $25/月（年繳 $20）、Premium seat $125/月（年繳 $100）；Go 價格不明（舊公告 $8）。官方說 Work 與 Codex 含在 Free、Go、Plus、Pro、Business、Edu、Enterprise 方案內，但 Free/Go 的 Work 僅限桌面 app，且用量有限。
**平台**：web、mobile（雲端 Work）、desktop app（macOS/Windows，可用本機檔案與桌面 app）。

## 2. 定位與使用場景

- 官方描述（搜尋摘要）：Work「is an agent designed for longer, multi-step work and finished deliverables」，可研究分析、跨已連接 app 與檔案作業，並產出文件、試算表、簡報、報告、Sites；可跟隨、提問、改方向、批准重要動作。可完成部分需登入的網站任務。
- 雲端 browser 範例（help 摘要）：查餐廳空位、找租屋房源。
- 前身 agent 範例（舊發表文）：研究、填表、編輯試算表，單次任務約 5–30 分鐘。
- Codex 才是軟體開發主場；Chat 為快速對話。三者在同一 app。

## 3. 執行環境

- **Work on web/mobile = 雲端**（help 摘要）。其「cloud container」不可存取你電腦的檔案或工具；OpenAI 安全概述稱「cloud tasks don't automatically get local files, browser sessions, or passwords」。
- **Work on desktop = 可用本機檔案與桌面 app，需你授權**；使用者「開啟資料夾或專案、僅授予任務所需檔案」。
- **Local work sync**：桌面啟動的任務可在 web/mobile 續接，步驟可用已連線電腦上的核准檔案與工具，但任務由雲端協調；「Local execution does not mean that conversation content... stays only on that computer」；不支援 Zero Data Retention。電腦離線時新一輪退回雲端 container。
- 前身 agent：雲端虛擬電腦，工具含 visual browser、text browser、terminal、code interpreter、connectors（舊發表文／help，摘要）。
- Cloud browser：預設「新網站先問」；設定有 Always ask／Auto approve／Always allow（官方不建議最後一項）；逐站 allow/block 覆寫。

## 4. 權限與安全模型（重點）

**本節大多出自 2025 年 agent 資料（system card、launch post）與 2026 年 Work help 文章的摘要；二者適用範圍不同，請勿混為一談。**

- 連接 app 的核准（Work help，聲明）：預設 "Important actions"——「allows reading from apps automatically but asks before actions that may have a meaningful effect outside ChatGPT, expose sensitive information, or be difficult to undo」；受管工作區由 admin 控制持久核准設定。
- 網站權限與動作核准是分開的：「Website access permission is separate from approval for consequential actions」（cloud browser help 摘要，聲明）。
- 登入：憑證走安全登入流程而非聊天；呈現登入請求前，另有 review model 檢查是否像釣魚（聲明）。發表文稱「username 與 password 不會被模型看到或用於訓練」（聲明，摘要）。卡住時請你 takeover cloud browser（手機或電腦），也可索取 takeover 連結。
- 前身 agent 機制（2025，聲明）：關鍵動作（如寄信、購物）前暫停確認；**watch mode**：敏感網站需你保持該分頁在前景，離開即暫停；**takeover mode**：登入時暫停讓你操作虛擬瀏覽器，期間不擷取截圖；terminal 網路限制；agent 時停用 memory。
- Prompt injection（聲明）：訓練、自動 monitors 與 filters、使用者確認、watch mode、terminal 網路限制；另有 **Lockdown Mode** 嚴格限制 ChatGPT 與外部系統互動以降低 injection 外洩。Help 文章說明攻擊者可藉網頁內容「exfiltrating data... taking actions the user didn't intend, or simply providing the user an incorrect answer」。
- 廠商自承限制：system card 的 injection 數據「test only the model's behavior, not our full end-to-end stack」；cloud browser help 稱已測試 prompt injection、phishing、非預期動作，但「safeguards do not eliminate every risk」，要求使用者自行檢視敏感步驟與重要結果；建議停用不必要的 app、避免敏感登入。
- 連接器風險：Help 提醒連接 app 時小心暴露敏感資訊。
- Admin：Work Local 與 Work Cloud 分開授權；Agent Security 區管理政策；MDM 對本機執行優先；Compliance API 可看 Work 對話與動作（聲明）。
- 資料保存：未能取得 Work 專屬保存細節（搜尋「virtual computer deletion」無結果），**未驗證**。Saved memories 與聊天記錄分開處理，刪除對話不一定刪記憶（摘要）。
- 矛盾線索：社群回報稱 Work 缺少安全登入交接／user takeover（使用者貼文，非官方），與官方 cloud browser 有 takeover 的說法衝突，需實測。另一則社群 bug（2026-10）稱 macOS cloud Work 在「Full Access」下無法寫入 Desktop，且「Ask for approval」在 cloud 模式灰掉（單一使用者回報，未驗證）。

## 5. 復原與可稽核性

- 可見/可停：官方稱可跟隨、提問、導向；舊 agent 可隨時 takeover 或暫停。
- 復原：未見 undo 機制的文件描述（**未驗證**）；唯一的防線是高影響動作先確認。
- 稽核：admin 層有 Compliance API；個人使用者的 action log 未見描述（未驗證）。

## 6. 記憶、排程與主動行為

- 排程：Scheduled tasks 可單次、重複或以事件觸發、監控變化；**事件觸發任務**來源為 Gmail、Slack、GitHub，每 30 次/小時、720 次/天；活動任務數上限（Free/Go 3、Plus 5、Business/Edu 10、Pro/Enterprise 15）；Enterprise 需 admin 開啟；任務可分享，但不轉移聊天記錄、本機檔案或記憶；桌面 app 可看但不能建立/編輯事件觸發條件（help 摘要）。
- Memory：有，且有 admin 角色層控制；agent 前身曾於執行時停用 memory 作為安全緩解。
- 這比 Cowork 更偏「主動」：事件觸發可在新信件、Slack 訊息、PR 變動時自動啟動，對安全面是額外攻擊面（我們的推論，非廠商說法）。

## 7. 擴充與寫程式能力

- Plugins：打包 skills 與連接的 apps（MCP）；Chat、Work、Codex 共用；plugin hooks 在雲端編排的 Work 不支援；Developer mode 與 MCP apps 為另頁（beta）。Admin 可控哪些 plugin／工具可用。
- 產出：文件、試算表、簡報（有專屬 help 文章）；**ChatGPT Sites**（公開 beta）：由 Work 或 Codex 生成託管網站或輕量 app，預設私人、預覽後部署；建置計入 Codex 用量，託管用量另計；EEA/瑞士/英國暫不提供公開發布。
- Codex：開發者路徑，可建立並部署輕量全端 JS/TS 網頁 app（Enterprise/Edu preview 自 2026-06-02）。

## 8. 成本與用量限制

- Work 與 Codex 共用同一份方案用量額度；非固定訊息數。Plus 有 5 小時視窗，估計本機訊息約 15–160／5 小時（依模型），並可能有週限；Pro 檔位無 5 小時限制但有週額度（Pro $100 為 Plus 的 5 倍、$200 為 20 倍）；超額可用 credits（Plus/Pro 可買）。（摘要，均未驗證）
- 舊 agent：Plus 40 則／月、Pro 400 則／月，僅初始請求計數，排程亦計（help 摘要）。Rate card 曾列 agent 約 30 credits／則。
- 新使用者回報大型任務可耗掉大半視窗（第三方，見 §10）。

## 9. 上手體驗（非技術使用者）

- 文件層：統一 app，在 Chat／Work／Codex 切換；首次需逐一授權資料夾、網站、app；預設 Important actions 與「新網站先問」對新手友善。
- 疑慮：提示可能過多（逐站、逐動作）造成 approval fatigue，其反面 "Always allow" 又被官方標為不建議；模式命名在一年內改了三次（Operator→agent→Work），說明文件互相矛盾，新手難找到正確指引。

## 10. 已知問題與外部評價（第三方，次要）

- TechCrunch 8 月實測（經 TechRepublic 轉述）：可完成真實任務，但雲端硬碟權限令人困惑、Calendar 只能建事件不能建新日曆。
- UseCarly：Gmail connector 僅 web 可寄、無附件、EU/UK 之外、每封需核准；Calendar 唯讀（可能已變動）。
- XDA：Work 比 Cowork 起步更順、較少 check-in。DataCamp、Composio：兩者互有勝負；Composio 稱 Cowork 展示更多計畫，Work 強調成品與執行軌跡。
- 社群：agent mode 被移除且無等價替代、menu 消失等（OpenAI Developer Community，使用者貼文）。
- 這些皆未驗證，且部分已過時。

## 11. 對我們的啟發

**Table stakes**：
1. 連接器動作分級：讀取自動、有外部影響/難復原才問（Important actions 模型）。
2. 網站存取核准與「重大動作核准」分兩層，互不覆蓋。
3. 登入／付款交給使用者接管（takeover），且接管期間不錄影截圖。
4. 敏感網站 watch mode（人必須在場）。
5. 排程任務與事件觸發任務要有上限與頻率限制。
6. 誠實聲明 injection 防禦不完整，並提供 Lockdown 式的嚴格模式。

**可差異化**：
- 命名與文件穩定性：OpenAI 一年內三度更名、help 文章自相矛盾，開源專案可用「單一、版本化、可 grep 的安全模型文件」取勝。
- 本地執行真的留在本地（OpenAI 的 local sync 仍由雲端協調、不支援 ZDR）。
- 使用者可讀、可匯出的 action log 與一鍵回滾（OpenAI 文件未見）。
- 事件觸發預設關閉，由使用者逐項啟用；對外寄信預設只存草稿。
- 用量透明：在任務開始前顯示預估耗用，而非共用黑箱額度。

## 12. 來源與未能驗證

**可達性**：help.openai.com、openai.com、cdn.openai.com、deploymentsafety.openai.com 於本環境皆無法連線（DNS 失敗／proxy 403）。以下僅為出現在 WebSearch 結果中的 URL，**我們沒有開啟過它們**：
- help.openai.com/en/articles/11752874（ChatGPT agent）、11794368（agent release notes）、20001275（ChatGPT Work and Codex）、20001280（cloud browser）、20001548（Agent Security and local work sync）、20001256（Plugins）、10291617（Scheduled tasks）、20001339（Sites）、6825453（ChatGPT release notes）
- openai.com/index/introducing-chatgpt-agent、openai.com/index/chatgpt-for-your-most-ambitious-work、system card（cdn.openai.com）
- chatgpt.com/pricing、learn.chatgpt.com/docs/pricing
- 次要：community.openai.com 貼文、techrepublic.com、datacamp.com、xda-developers.com、usecarly.com、findskill.ai、composio.dev

**未驗證清單**：
- 各方案的精確價格與 Work 額度；
- agent 是否仍可用（官方頁自相矛盾）；
- Work 專屬的動作核准清單、watch mode／takeover 在 Work 是否沿用；
- 資料保存期間與虛擬環境刪除；
- 底層模型名稱（搜尋結果出現 GPT-5.6 與 GPT-6 Astra 兩種說法，互相不一致）；
- 不得從記憶補充任何未在上列摘要中出現的事實。
