# Goose 研究報告

查證日期：2026-10-25。方法：以 GitHub 頁面、raw.githubusercontent.com 上的 repo 內文件原始檔（documentation/docs/**）、GOVERNANCE.md、releases 頁為主要來源。官方網站 goose-docs.ai 與 block.github.io 在本環境被 proxy 擋（DNS/403），故文件內容讀的是 repo 內 markdown 原始檔，並非渲染後網站。未讀 Rust 原始碼本體，涉及實作細節處標註「文件所述」。

## 1. 基本資料
- Canonical repo（verified）：**github.com/aaif-goose/goose**（原 block/goose；README 與文件連結皆已指向 aaif-goose）。block/goose 是否自動 redirect：unverified。
- License：Apache-2.0（verified）。
- 語言：README 稱 "Built in Rust"；GitHub 語言比例本次未取得（unverified）。crates/ 目錄含 goose、goose-cli、goose-mcp、goose-agent、goose-providers、goose-context-management、goose-local-inference、goose-sdk、goose-acp-macros 等（verified）。桌面 App 的 UI 技術（文件的 lucide-react 只是文件站；Electron/TypeScript）本次未直接驗證。
- 治理（verified）：README 稱 goose 屬 Linux Foundation 旗下 **Agentic AI Foundation (AAIF)**；repo 內 GOVERNANCE.md 為輕量模型：Contributors / Maintainers / Core Maintainers；日常以共識，僵局由 goose 創建者 Bradley Axen 擔任 tie breaker；重大變更需公開審查至少一週並經 Core Maintainers 多數同意。仍保有「創建者決勝」條款，並非完全中立的 foundation 治理（推論）。
- 資金：未驗證（AAIF 會員/出資結構本次未查）。
- 活躍度：約 55.7k stars、6.4k forks、352 open issues（GitHub 頁面）。Releases：v1.56.0（2026-10-23）、v1.55.0（10-16）、v1.54.0（10-08）——約每週一版，非常活躍。
- 另有 Custom Distributions（CUSTOM_DISTROS.md）：可 fork 出預設 provider/extension/品牌的發行版——對我們是直接競品訊號。

## 2. 架構
- 核心：Rust agent（crates/goose），CLI、桌面 App、API/SDK 共用同一 core；Desktop 與 CLI 共用同一 session database，可互相 resume（文件 verified）。
- 另有 remote-goose-server、ACP（Agent Client Protocol）支援：goose 可作 ACP agent，亦可用 ACP provider 借用 Claude/ChatGPT/Gemini 訂閱或 codex-acp（README/文件 verified）。
- Extension 全面 MCP-native：內建 extension「本身就是 MCP server」（文件原話意）。載入方式：Desktop 設定頁/ deeplink、CLI `goose configure`、`--with-extension`（stdio 指令）、`--with-builtin`、session 內 `/extension`、`/builtin`；亦有 Extension Manager 平台 extension 讓 agent 於 session 中自行搜尋/啟用 extension（verified）。
- v1.55 changelog：移除舊 agent loop、背景 subagent 與 orchestrator；v1.56：Boa JS runtime 取代 PCTX 作 Code Mode。代表架構仍在大幅重構中（verified 自 release notes）。

## 3. 工具集與 extensions
- 預設啟用 Developer（shell、edit、tree 等，另有 codebase analysis；文件標 shell/edit 為 High risk）。
- 內建（可選）：Computer Controller（網頁抓取、檔案快取、macOS UI 自動化 via Peekaboo、PDF/Word/Excel 處理，定位「不需寫程式」）、Memory。
- Platform extensions：Chat Recall、Extension Manager、Skills、Summon（載入知識來源＋委派 subagent）、Todo、Top of Mind（每輪注入持久指令）、Code Mode。
- 第三方：任何 MCP server（npx/uvx 指令、SSE/HTTP）；README 宣稱 70+ extensions。MCP elicitation、roots 有專頁。
- 效能警示（文件 verified）：啟用工具總數宜少於 25；Code Mode 以 JS 動態發現/批次呼叫工具來緩解 context 負擔。
- Extension Allowlist（GOOSE_ALLOWLIST，YAML URL）供企業限制可安裝的 MCP。

## 4. Recipes / sub-recipes / scheduling
- Recipe = 可分享的 workflow 封裝（YAML：instructions/prompt、parameters、extensions、settings、可能含 response schema 等；細節見 recipe-reference，未逐項驗證）。可由當前 session 產生、一鍵（deeplink）啟動、存於本地/GitHub repo（storing-recipes）、並可綁自訂 slash command。
- Sub-recipes（標示 **experimental**）：主 recipe 以 `sub_recipes` 註冊，每個轉為一個 tool，在隔離 session 執行（不共享歷史/記憶），不可巢狀；注入 summon 的 `delegate` tool。
- Scheduling：Desktop 有 Scheduler 視圖，從 Recipe Library 建立排程，頻率 UI 轉成 cron；另有 `goose run`（headless）可接系統 cron。
- 對非開發者：Recipe Library + GUI 排程 + 參數化是明顯優勢；但撰寫 recipe 仍偏 YAML/CLI（reference 為 CLI 導向），GUI 內是否能完整編輯：部分 verified、完整度 unverified。#10113 顯示 recipe metadata 不完整時 session 無法載入，屬穩定性問題。

## 5. Context 管理
- Auto-compaction：預設達 model context 的 80% 或 225,000 tokens（取小者）觸發，摘要舊對話；可用 GOOSE_AUTO_COMPACT_THRESHOLD 調整、0.0 關閉；compaction prompt 可改（compaction.md 範本）；Desktop 可手動 "Compact now"，並有 token 用量指示點。舊 tool output 會在背景摘要（GOOSE_TOOL_CALL_CUTOFF）。
- `.goosehints`：全域（~/.config/goose/.goosehints）+ 專案層，衝突時專案優先；**需啟用 Developer extension 才會讀**（文件 verified，是個耦合點）。
- Memory extension（記偏好）、Top of Mind、Chat Recall（跨 session 搜尋）、Skills。
- Sessions：多 session 並行、命名、搜尋、resume、duplicate、fork（`--resume --fork`、編輯訊息建立 fork）、HTML export（v1.54）。文件自己建議新任務開新 session 以避免 "doom spiraling"。

## 6. 權限與安全
- 四種模式（verified）：Completely Autonomous（**預設**）、Manual Approval、Smart Approval（風險導向自動核准低風險）、Chat Only。每 session 各自模式；CLI `/mode`。v1.54 增加首次設定時的 mode 提示。
- Tool 粒度：Always Allow / Ask Before / Never Allow，可逐 extension 的逐 tool 設定（僅 Manual/Smart 模式）。粒度只到「tool」層級；#11399（open，15 則留言）要求對 shell 命令做 regex allow/ask/deny——證實**目前沒有命令層級規則**。
- Prompt injection detection：pattern 比對（rm -rf、下載執行、讀 SSH key 等）+ 可選 ML classifier（自備 endpoint，如 Hugging Face）；有信心分數與閾值、Allow Once/Deny、記錄 finding ID。文件自承「safeguard, not a guarantee」。
- Adversary Mode：放 ~/.config/goose/adversary.md 即啟用，獨立 agent reviewer 在每次 tool call 前看原任務+近期訊息判斷 ALLOW/BLOCK；**fail-open**（reviewer 失敗則放行），用同一 model。
- Extension 安全：allowlist；known-issues 有 "Malicious Package Detected" 條目（機制細節未讀，unverified）。
- 預設 autonomous + shell 為 user 權限：對新手是高風險預設（文件自承）。

## 7. 復原機制
- 文件中**未找到** undo / checkpoint / rewind / 檔案快照功能（grep 全部已下載文件無結果）。known-issues 的建議是「用 git 暫存自己的修改、把 goose 的編輯分開 commit 以便 revert」。
- 有的是 session 層級：fork session、編輯訊息從該點分叉、interrupt。這是對話復原，不是檔案復原。
- 結論：檔案層級復原＝依賴使用者 git（verified 文件態度；程式碼層面是否有未文件化功能：unverified）。刪除 session 不可復原。

## 8. 模型與成本
- Provider：15+（Anthropic、OpenAI、Google、Ollama、OpenRouter、Azure、Bedrock、Databricks、GitHub Copilot、Tetrate Agent Router 等）；ChatGPT Plus/Pro 訂閱（browser OAuth）；ACP provider 借用 Claude/Gemini/Codex 訂閱。
- 本地：Ollama、LM Studio、Docker Model Runner、Ramalama、Atomic Chat；repo 另有 goose-local-inference、goose-download-manager crates（內建本地推論，功能完整度 unverified）。有 "tool shim" 讓不支援 tool calling 的本地模型可用（tool-shim.md 存在，細節未讀）。
- 新手引導：文件「建議新使用者從 Tetrate Agent Router 開始」，首次 OAuth 贈 $10 credits（商業合作導流，應留意）。
- 成本顯示：只有 CLI 的 GOOSE_CLI_SHOW_COST 環境變數（預設關）；Desktop 有 token 用量指示，**Desktop 的金額顯示未驗證**。
- Lead/worker：舊版曾有 lead/worker；目前 multi-model 文件改列 `/model` 中途切換、subagent 各自 provider/model、context engineering。本次在現行文件**未找到** lead/worker 設定（視為已移除或改名，unverified）。
- Prompt caching：Anthropic 等自動啟用。

## 9. 上手體驗
- 安裝：macOS/Windows/Linux 桌面 App（zip、Homebrew cask `block-goose`、v1.56 新增 Windows NSIS installer）；CLI 為 curl script。
- 首次執行：Desktop 有 onboarding provider 設定元件；一鍵 OAuth（Tetrate/ChatGPT）免貼 API key，對新手友善。首次詢問匿名使用資料收集（opt-in，不含對話/程式碼）。
- 已知摩擦（known-issues.md verified）：Apple M3 無視窗需修 ~/.config 權限、Keychain/keyring 錯誤、Windows Node extension 無法啟動、公司 proxy、WSL 連 Ollama、Hermit 錯誤、rate limit、context length exceeded、"stuck in loop"。
- 誠實評估：安裝與 provider 設定對非工程師屬同類最佳之一；但預設 autonomous 且給 shell 全權限、需理解 extension/MCP/mode 等概念、錯誤訊息多指向 Discord/文件。
- 寫程式表現：定位是通用 agent，coding 靠 Developer extension（shell/edit/tree + code analysis）。edit 為精確字串取代；無 undo；無原生 diff/review 層（文件未見，unverified）。品質高度取決於所選 model。

## 10. 優缺點與痛點
優點：Apache-2.0 + 中立基金會；Desktop+CLI+API 同核心；MCP 原生、ACP 兩向整合；Recipes+排程+deeplink 分享；多層安全（injection 偵測、adversary、allowlist）；provider 與本地模型選擇廣；發版快。
缺點：無檔案層級 undo；預設最寬權限；權限粒度止於 tool（#11399）；sub-recipes 實驗性；API/架構快速重構（v1.55 刪舊 loop、移除 subagent orchestrator）導致 recipe/文件穩定性風險；hints 依賴 Developer extension；工具過多效能下降（<25 建議）；Adversary fail-open。
新手痛點：概念多（extension/mode/recipe）；危險預設；平台特定錯誤（keyring、M3、Windows）；long session 會 "doom spiral"，需自己懂得開新 session；無金額可視化（Desktop 未驗證）。
專業痛點：需 shell regex 規則（#11399）；Recipe 在 metadata 不全時無法載入（#10113）；sidebar session 管理（#10277 pin/rename/archive）；ACP client 設定別名（#12255）；開 issue 總量 352。
（Issue 資料來自 GitHub issue 列表頁；留言數為頁面所顯示數字，反應數未取得。）

## 11. 對我們的啟發
值得抄：
1. 一鍵 OAuth onboarding（免 API key）+ 預設推薦路徑。
2. Recipe 概念：可分享、可參數化、deeplink 啟動、GUI 排程——但要做到 GUI 內可完整建立/編輯（Goose 偏 YAML）。
3. Smart Approval（風險分級）+ per-tool 規則；Chat Only 模式適合新手。
4. 自動 compaction 並可視化 token；session fork/edit-from-message。
5. Custom distribution 機制（預設包）；Extension Allowlist。
6. Adversary 式獨立審查（但建議 fail-closed 或至少可設定）。
避免：
- 預設 autonomous + 全權限 shell；
- 沒有檔案 undo（我們可內建 git checkpoint / 自動 shadow commit，作為明確差異化）；
- 權限只到 tool 層級，應做到命令/路徑層級規則；
- 功能與 extension 太多導致 tool 數爆量（需預設精簡 + 動態載入）；
- hints 與特定 extension 耦合；
- 在快速重構中頻繁破壞 recipe 格式，需 schema 版本化；
- 引導使用者至單一商業 router 的利益衝突觀感。

## 12. 來源與未能驗證
來源（皆於 2026-10-25 取得）：
- https://github.com/aaif-goose/goose （README、issues 列表、releases 頁、crates 目錄）
- https://raw.githubusercontent.com/aaif-goose/goose/main/README.md、GOVERNANCE.md
- 同 repo documentation/docs/ 下：getting-started/{installation,using-extensions,providers}.md；guides/managing-tools/{goose-permissions,tool-permissions,code-mode}.md；guides/security/{index,prompt-injection-detection,adversary-mode}；guides/recipes/{index,subrecipes,session-recipes,recipe-reference,storing-recipes}；guides/sessions/{session-management,smart-context-management}.md；context-engineering/using-goosehints.md；guides/{environment-variables,allowlist,usage-data,tool-shim,running-tasks}.md；troubleshooting/known-issues.md；mcp/{developer-mcp,computer-controller-mcp}.md；guides/multi-model/index.mdx

未能驗證：
- goose-docs.ai 渲染後官網（proxy 擋）；block/goose 的 redirect 行為；GitHub 語言比例；桌面 App UI 技術棧。
- 資金/AAIF 出資與 goose 加入 LF 的日期、Block 的後續角色。
- 原始碼層級：實際 undo 功能是否存在、Smart Approval 的風險判定實作、local inference 品質、Desktop 是否顯示金額。
- lead/worker 現況（現行文件無，可能已移除）。
- issue 的 reaction 數與其他社群討論（Discussions、Reddit、HN）未查；痛點多源自官方 known-issues 與 4 個 open issues，樣本小。
- recipe YAML 完整欄位與 GUI 編輯能力；Malicious Package Detected 機制細節。
