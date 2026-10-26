# Vellum Assistant 研究筆記

查核日期：2026-10-26。標記說明：【已驗證】= 來自官方 repo 原始檔或官方 docs 原始碼；【vendor】= 廠商自述，行為未獨立驗證；【二手】= 部落格或目錄站；【推論】= 我方判斷。

## 1. 基本資料

- **Repo**：github.com/vellum-ai/vellum-assistant【已驗證】。
- **License**：MIT，Copyright (c) 2025 Vellum AI【已驗證，LICENSE 檔】。
- **Stack**：TypeScript / Bun 為主。頂層目錄有 `assistant`（daemon）、`gateway`、`credential-executor`（CES）、`cli`、`clients`（web、iOS、Android、macOS、Windows，桌面是 Electron）、`packages`、`plugins`、`skills`。Memory 使用 SQLite + Qdrant sidecar。【已驗證，README 與 ARCHITECTURE.md】
- **公司與商業模式**：Vellum AI 是營利公司。README 寫明開源 assistant 免費、MIT，另有 managed 的 Vellum Platform「sustains the business」。定價頁有 Base（免費）與 Pro（Mighty $30/月、Super $100/月、Ultra $200/月，或 Custom），另有 prepaid credits。【已驗證，README 與 docs 原始碼】
- **與公司其他產品的關係**：vellum.ai 過去賣的是企業 LLM 工作流平台（visual workflow builder、SDK、eval、版本化部署）。二手資料稱 2026 年 5 月轉向 personal assistant【二手，目錄站與搜尋摘要，我沒看到主要公告，轉向日期和投資人名單不確定】。Repo 的 Constitution 寫於 2026-05-11，與此時間吻合。PyPI 上的 `vellum-ai` 是舊平台 SDK，和本 assistant 是不同產品線，不要混淆。
- **活躍度**：README 頁顯示約 1.4k–1.5k stars、219 forks、`main` 上 30,757 commits。這個 commit 數對一個 2025 年才有 LICENSE 年份的專案來說很高，顯示極密集的（很可能大量由 AI 輔助的）開發【推論】。Releases 頁最新是 0.12.6（9/29），9 月內共 10 個 release，約每 3 天一版；我看到的頁面可能有快取，10 月的實際最新版本未驗證。Open issues 56 個，最新到 10/19。
- **規模評價**：這不是早期小專案。這是一個大型、快速演進的 monorepo，但 Windows 版仍是 dev build，Docker 與 Apple Container 的本地隔離選項標為「Coming soon」。

## 2. 定位與使用場景

- 目標使用者：Constitution 明說「Every human」，不是給開發者的工具；早期使用者偏技術，訊息會隨成熟度調整（Article II）。README 直接對標 OpenClaw、Hermes Agent、Claude Code，主張「一次下載就能用，不用自己 hatch 好幾次」。【已驗證】
- 用途：個人助理（email、行事曆、Slack、Telegram、電話、每日簡報、訂餐/購物）、電腦操作（computer use）、記憶長期個人化、也能建小型 web app。
- 產品語言偏「關係」而非工具：assistant 有自己的名字、email、帳號；使用者稱為 guardian。

## 3. 架構

- **兩種跑法**：Managed（Vellum Cloud，預設且被推薦）與 Local（Mac/Windows 本機原生 process）。另有 user-hosted remote（自己的 GCP 或 Mac Mini）。同一份 codebase。【已驗證】
- **元件**：daemon（assistant runtime，HTTP + SSE）、gateway（唯一公網入口、風險分類、trust rules、閾值）、CES（獨立 process 的憑證服務）、clients。桌面端是 Electron 殼，macOS 另有 native sidecar。
- **Channels**：macOS、Windows、iOS、Android、Web、Voice、Email、Telegram、Slack、Twilio 電話。一個 assistant 一份記憶。新 channel 需 6 位數 guardian 驗證握手。
- **狀態儲存**：本機在 `~/.vellum/workspace/`，內含 `IDENTITY.md`、`SOUL.md`、`USER.md`、`NOW.md`、`config.json`、`skills/`、`data/db/assistant.db`（SQLite）。docs 強調「everything is plain text」，可用任何編輯器看、可放版本控制。【已驗證】
- **模型插拔**：Anthropic（主要）、OpenAI、Gemini、Fireworks、OpenRouter、MiniMax、任何 OpenAI-compatible endpoint、Ollama 本地模型。Embeddings 預設本機 ONNX。【README 與 docs】
- **Linux**：沒有桌面 client，只有 web、Chrome extension 或自架 runtime。

## 4. 工具與整合、Skills、MCP

- 內建工具：bash（workspace 沙箱內）、`host_bash` / `host_file_*`（真實主機，需核准）、file、web fetch、browser（headless 與 Chrome extension）、computer use（macOS Accessibility、Windows UI Automation）、screen watch、subagent。
- Skills：`skills/` 目錄有 76 個子目錄（Gmail、Calendar、Slack、Notion、DoorDash、Amazon、Twilio、plugin-builder、mcp-setup 等）；docs 說出貨 28 個 bundled skills。Skill 由 `SKILL.md` + `TOOLS.json` 組成，可從 catalog 安裝、放進 workspace、或由 assistant 自己建立。【已驗證】
- 「Self-improving skills」：完成工作後，memory 會把實際執行過的流程存成可重用 skill；docs 明說這不是重訓模型。使用者可檢視、編輯、刪除。
- MCP：docs 列「Add MCP servers」，repo 有 `mcp-setup` skill。但安全文件寫明「一個 user rule 目前無法涵蓋 MCP 或其他沒有 classifier 的工具」，MCP 工具走 fallback 風險等級（registry default 或 medium）。Issue #42552「Unify Integrations and add a standard MCP catalog」仍 open，表示 MCP 體驗還在整理中。
- OAuth：Slack、Notion、Google、HubSpot、Linear、Discord、Twitter 等。

## 5. 記憶與主動行為

- README 宣稱 8 種記憶類型（episodic、semantic、procedural、emotional、prospective、behavioral、narrative、shared）【vendor】。但 `assistant/docs/architecture/memory.md` 與 docs 站描述的是另一套：concept-page「wiki」，markdown 文章放在 `memory/concepts/`，搭配 Qdrant 的 dense + BM25 sparse 檢索，新 workspace 預設 v3，v1、v2 正在退場。**README 與目前的實作描述並不一致**，README 的「8 types」可能已過時或屬行銷用語【推論】。
- 使用者可問「你記得我什麼」，可說「忘記某事」，有 memory map 視覺化，可手動 Create memory。Docs 也警告：敏感資訊進入對話後可能變成 memory 並進入之後的 model call。
- 主動行為：README 稱每小時重讀自己的筆記，找未完成或將到期事項並通知（heartbeat）【vendor；有 `vellum-heartbeat` skill，但行為細節我未驗證】。背景任務預設風險閾值為 Relaxed，比互動模式寬鬆。
- 磁碟 95% 時進入 cleanup mode，暫停背景工作與遠端輸入【已驗證，ARCHITECTURE.md】。

## 6. 威脅模型與權限

**【已驗證】的設計**（`assistant/docs/architecture/security.md`、docs 站）：

- 風險分類在獨立的 gateway process 做，不由模型判斷。Shell 指令用 tree-sitter 解析。工具分 Low / Medium / High。Gateway 不可達時 fail closed。
- 使用者可見四檔 risk tolerance：Strict、Default、Relaxed、Full access。**預設 Default**：Low 自動，Medium 與 High 要問。排程任務預設 Relaxed，API/webhook 觸發預設 Strict。
- 沙箱邊界：workspace 內（讀寫檔、bash、建 app）免核准；主機端（`host_file_*`、`host_bash`）每次要核准。路徑穿越與 symlink 逃逸有擋。容器模式才有 OS 層隔離；**原生本機模式沒有 OS 層隔離**，docs 自己也說原生是「full access to your system」。
- 提示畫面：白話描述、紅黃綠風險標籤、風險原因、可展開完整輸入。Allow 會順便建立 trust rule；「Allow & Create Rule」可先編輯規則。拒絕後不會自動重試。提示逾時視為 deny。
- 第三方 skill 工具預設一律要問，不論風險；trust rule 綁 skill 內容 hash，skill 被改就重新詢問；寫入 skill 原始碼被分類為 High。
- 憑證：CES 獨立 process，模型看不到明文；有 broker 的 tool/domain policy；使用者訊息入口有 secret 偵測，整則看起來像 token 的訊息預設被擋。
- 陌生人：未驗證的外部聯絡人得到寫死的拒絕訊息，不經模型。受信任聯絡人要做敏感動作，需 guardian 逐次核准（一次性，5 分鐘過期）。
- Computer use：每步單獨提示、50 步上限、loop 偵測、危險快捷鍵被擋。

**誠實的限制與矛盾**：

1. Docs 寫「沒有辦法 prompt-inject 繞過權限邊界」，這是強主張【vendor】。實際上它只涵蓋「有被分類並進入 gateway 的 tool call」；一旦使用者核准、或使用者設了寬鬆閾值，模型被注入後能做的事就是被核准的範圍。Docs 提到的 untrusted content 包裝（`<external_content>`、never-follow 語意）只在 watch/screen 流程看到，是否普遍套用於網頁、email 內容我**未驗證**。
2. CES 的隔離只擋「注入」，不擋「輸出」：stdout/stderr 會回給 assistant，指令若 echo 出憑證就會進入模型上下文；`host_bash` 在強機密保證之外（`assistant/ARCHITECTURE.md`，官方自己寫的）。
3. **Permission controls v2**（feature flag）：ARCHITECTURE.md 寫明，在此旗標下只剩「主機電腦存取」是確定性核准，其他工具改為「model-mediated consent」，不再有逐工具核准卡。這與 docs 站「全部確定性」的說法方向相反。旗標預設值我未驗證，但方向值得注意：產品正在往減少核准摩擦走。
4. 官方 docs 自己警告「autopilot」：點幾十次 Allow 後會反射性同意。
5. SECURITY.md 很短：私下通報 security@vellum.ai、48 小時內回覆、5 個工作天給時程。沒有獨立安全稽核報告可查【未驗證有無】。
6. Skill 市集：docs 說「Skills are audited」，但稽核流程未見說明【vendor】。

## 7. 復原與可稽核性

- 每次工具呼叫都寫一筆 `tool_invocations` 稽核紀錄（決策、風險等級、遮蔽後輸入、結果預覽、時長）【已驗證】。Constitution 把「irreversible action without informed consent」列為要防的失敗模式。
- Workspace 是純文字檔，使用者可自己放進 git。ARCHITECTURE.md 提到 activation 流程有「turn-boundary Git commit」，暗示 workspace 每回合會自動 git commit，但我沒找到專門文件，**其復原用法未驗證**。
- `vellum retire` 會封存 workspace；雲端可 reset workspace。`vellum backup` 對 Vellum Cloud assistant 失敗（issue #43478）、從 Cloud teleport 失敗（#43477），表示備份/搬移這條路目前不穩。
- 沒有看到「回到某次 agent 修改前」的使用者級 undo UI【未驗證】。

## 8. 模型與成本

- 兩種連線方式：Sign in with Vellum（managed，Vellum 代付 API key）或 BYOK（自架，用自己的 Anthropic 等 key）。
- 計費：Vellum 宣稱 token 成本原價轉嫁、無加價【vendor】。收費在機器規格、儲存、平台費（$10/月）與 usage bundle。包含用量每月重置不累積，prepaid credits 12 個月到期，先花包含用量再花 credits。
- 成本可見度：Billing 頁顯示 Credit Balance、Auto-Reload、**每日 credit 上限**、低餘額警示，Usage 分頁按天、模型、動作拆分。Conversation 表格存有 token 數與估計成本。這比多數開源 harness 好。
- 上限：managed 超額會回 `MANAGED_USAGE_LIMIT` 錯誤。
- 已知痛點：#43465「Managed inference profiles 卡在 GPT-5.6，無法編輯或隱藏」。

## 9. 上手體驗（桌面 app 第一次執行）

以下全部來自官方 docs 原始碼【已驗證】，但我沒有實際安裝，畫面細節未實測。

1. 到 vellum.ai/downloads 下載 macOS `.dmg`（需 macOS 15+，約 500 MB）或 Windows `.exe`（Windows 10+，x64 或 ARM64；目前是連到開發環境的 dev build）。macOS 版有簽章與 notarize，Windows 有 Authenticode 簽章。
2. Mac 上把 app 拖到 Applications 打開；Windows 跑安裝程式。「不需要終端機」。
3. 先註冊 Vellum 帳號，設定隱私偏好並接受服務條款。
4. 登入後，**預設連到雲端 assistant**（大約 30 秒 hatch 完）。若要純本機，要在 Local 模式另外選。README 描述為「Pick your mode：Managed 或 Local」，使用者要決定的第一件事就是資料放哪裡。
5. 沒有 onboarding wizard，「就是一個聊天框」。Docs 建議第一句話：幫它取名、介紹自己、問「你會什麼」、調整語氣。助理一開始是空白的（沒名字、沒記憶）。不過 ARCHITECTURE.md 顯示背後有 OnboardingOrchestrator 與首次引導 playbook、以及 activation checklist（一組「新手任務」清單與進度 pill），所以實際畫面可能比 docs 說的更有引導，細節未驗證。
6. 權限不是一開始全部要，而是用到才要：Screen Recording、Microphone、Speech Recognition、Accessibility、Notifications 各自在「第一次需要時」才跳 macOS 系統提示。Docs 特別說 app 沒有要 Full Disk Access，個別檔案與 shell 動作仍走 app 內核准。
7. 第一次出現會有影響的動作（寄信、存檔、排行程），會停下來顯示做什麼，使用者按 Allow 或 Deny。
8. 之後再選擇性連 Telegram、Slack、電話、email，每個 channel 需要 6 位數驗證。更新時右上角出現綠色更新鈕，可立即或稍後。

非技術使用者要面對的決策：雲端或本機、要不要付費方案、要不要給 Accessibility/Screen Recording、風險等級（預設 Default）、每個 Allow 的範圍（Allow 會建立 rule）。

## 10. 能否替使用者做出東西

- 有 App Builder skill：「build interactive apps, dashboards, tools, data visualizations」，產物存在 `data/apps/`（app JSON 與頁面），在桌面 app 的 Dynamic Workspace（WebView）中顯示。docs 的範例提示：「Build me a simple expense tracker I can use in the browser」。另有 `deploy-fullstack-vercel`、`frontend-design`、`plugin-builder`、ACP（委派給外部 coding agent）、`headless-claude-code` skill。
- 自動化：schedule（cron、RRULE、一次性）、watcher、playbooks（觸發-動作規則）、workflow orchestration（assistant 自己寫 JS/TS，在 QuickJS-WASM 沙箱執行，只有 hooks，沒有檔案、網路權限；能力在每次 run 宣告時一次同意）。
- 品質與可維護性：產出的小 app 品質、能否長期維護，我沒有實測【未驗證】。

## 11. 優點 / 缺點 / 痛點

優點：
- 權限由獨立 gateway 做確定性分類，不靠模型自我約束；Fail closed；憑證與模型分離；陌生人硬拒絕。
- 風險分級用顏色與白話原因呈現，並有 Strict 到 Full access 四檔。
- Workspace 是人可讀的純文字；有 trust rule 編輯器；稽核表齊全。
- 費用可見（日上限、餘額、分項用量）。
- 文件透明度高，Constitution 與 docs 明講失敗模式。

缺點與痛點：
- 產品重心是 managed 雲端 + 付費方案；開源部分更像「可自架的客戶端與 runtime」。README 說「CLI works but desktop app is our primary focus」。
- 複雜度極高（30k commits、數十個服務與旗標）。對想理解並 fork 的團隊是負擔【推論】。
- 原生本機模式無 OS 隔離，Docker / Apple Container 隔離仍是「Coming soon」。
- README、docs、架構文件之間有落差（記憶 8 種 vs concept wiki；權限全確定性 vs v2 model-mediated）。
- Open issues 顯示的穩定性問題（截至 10/19，共 56 個 open）：#43453 Qdrant sidecar 在 cwd 為 `/` 時啟動 panic；#43479 `vellum pair` 在 rootless Docker 回 403；#43478/#43477 備份與 teleport 失敗；#42732 通知回報成功卻顯示「tool result missing」、桌面 push 沒出現；#43482 Deepgram live voice handshake 400。多半是整合與邊角問題，沒有看到公開的重大安全 issue（但安全問題本來就要走私下通報）。

## 12. 對我們的啟發

1. **桌面 onboarding**：他們選擇「零 wizard、只有聊天框、權限延後到用到才要」，認知負擔低。但代價是使用者的第一個真實決策（雲端或本機、Allow 的 rule 範圍）沒有被教學。我們可學「延遲授權」，但要在第一次核准時就用一句話解釋「Allow 會變成長期規則」。
2. **安全的呈現方式**：核准卡顯示「做什麼 + 風險色 + 為什麼這個風險 + 展開細節」，且有「Allow once / 建立規則」分流，是值得複製的模式。我們可以更進一步，把預設改成只給 once，建立規則要多一個步驟，來對抗 docs 自己警告的 autopilot 問題。
3. **「工作區是另一台電腦」比喻**：docs 用「電腦裡的另一台電腦」解釋 sandbox，非技術者容易懂。主機端動作才打擾使用者，是好的預設分界。
4. **要避免的**：把安全寫成絕對（「無法 prompt inject」）；原生模式預設給完整主機存取卻沒有 OS 層隔離；把核准全面改成模型自行判斷（v2 方向）。對 vibe coder 的目標，這兩點正好是我們可以做差異化的地方。
5. **成本顯示**：每日上限、分項用量、低餘額警示，建議作為預設而非選配。
6. **復原**：Vellum 的 undo 不明確。我們若做「每次 agent 改動都是可回滾的 git commit + 一鍵還原」，可補上這個缺口。

## 13. 來源與未能驗證

**主要來源**（皆於 2026-10-26 讀取）：
- github.com/vellum-ai/vellum-assistant 頁面（README、stars、releases、issues、skills 目錄）。
- raw.githubusercontent.com/vellum-ai/vellum-assistant/main/：`README.md`、`LICENSE`、`SECURITY.md`、`CONSTITUTION.md`、`ARCHITECTURE.md`、`assistant/ARCHITECTURE.md`、`assistant/docs/architecture/security.md`、`memory.md`、`clients/docs/src/app/docs/**` 下的 docs 頁面原始碼（installation、quick start、permissions model、best practices、pricing、hosting options、memory、skills、what-is-vellum）。
- vellum.ai 網域在本環境被 proxy 擋住（403），所以 docs 內容是從 repo 內 docs 站原始碼（JSX）取得，而非渲染後頁面。
- 二手：gittrend.io 與 notes.dsebastien.net（stars 與 repo 描述）、搜尋摘要中的公司背景（Series A $20M 2025-07、總計 $25M 等，投資人名單各來源不一致）。

**未能驗證**：
- 實際安裝與第一次執行的畫面（本研究是讀文件，未實測）。
- Permission controls v2 旗標的預設值，以及 Default 風險檔在目前版本是否仍是確定性核准。
- 每小時 heartbeat 的實際行為與使用者可控程度。
- Workspace 是否每回合自動 git commit，以及能否用來還原。
- 網頁、email 等不受信內容是否普遍有 prompt injection 包裝。
- Skill 市集「audited」的稽核流程。
- 10 月的最新 release 版本與發版日期（我看到的是 0.12.6，9/29，可能是快取）。
- 公司轉向 assistant 的確切日期與融資細節（僅二手來源）。
- App Builder 產出的實際品質。
