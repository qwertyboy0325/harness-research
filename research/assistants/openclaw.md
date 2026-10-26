# OpenClaw 研究報告

檢查日期：2026-10-26。
證據層級標記：**[P]** 為一手來源（repo 原始碼 / docs 原始 markdown / GitHub advisory 頁面）；**[S]** 為二手（新聞、安全廠商部落格、Wikipedia）；**[推論]** 為本報告推論；**未驗證** 表示無法確認。

方法說明：
- 我們 `git clone --depth 1 https://github.com/openclaw/openclaw`，取得 commit `c43d6382`（2026-10-25）。以下 [P] 引用皆來自該 checkout 的 `README.md`、`SECURITY.md`、`VISION.md`、`docs/**`、`package.json`，以及 WebFetch 讀取的 github.com 頁面。
- 官方文件站 openclaw.ai / docs.openclaw.ai 在此環境無法解析（`ENOTFOUND`），因此一律改讀 repo 內 docs 原始 markdown。
- `gh api` 對該 repo 被 scope 限制，advisory 資料是用 WebFetch 讀 github.com 頁面，WebFetch 以小模型摘要，數字（stars、頁數）可能有誤差。
- openclaw.ai 部落格（foundation 公告、VirusTotal 合作公告）無法讀取，相關內容僅能從二手來源得知。

---

## 1. 基本資料

| 項目 | 內容 |
|---|---|
| Canonical repo | `github.com/openclaw/openclaw` [P]；ClawHub（skill 註冊中心）為獨立 repo `github.com/openclaw/clawhub`（MIT，約 9.5k stars）[P] |
| 授權 | MIT，版權 "© OpenClaw Foundation"（README 末段）[P] |
| 技術堆疊 | TypeScript / Node.js（需 Node 24.16+ 或 26.1+，建議 26），pnpm monorepo；原生 App 為 macOS / iOS / Android / Windows / Linux；`crates/` 有 Rust 部分；狀態預設 SQLite，可選 PostgreSQL（`package.json` 的 `databaseEngines`）[P]。VISION.md 說明選 TypeScript 是為了「hackable by default」[P] |
| 建立者 / 組織 | Peter Steinberger（steipete）與社群 [P]。README：由 OpenClaw Foundation 管理，為獨立 501(c)(3)，Foundation 雇用核心團隊並簽署 release；捐助者不擁有或指揮專案；OpenAI 是 donor 而非 owner [P] |
| 資金 / 商業模式 | README：無付費方案、無託管服務、無 token；Foundation 由 Amazon、Lobster Computer Company、Offline Holdings、OpenAI、Red Hat、University of Michigan 等捐助，基礎設施由 Blacksmith、Convex、GitHub、NVIDIA、Vercel 支援 [P]。docs/start/why-openclaw.md 另列 Atlassian、Microsoft、Tencent 等「超過三十個組織」[P]；SECURITY.md 提到維護者包含 NVIDIA、Tencent 的工程師 [P]。VISION.md 提到「託管服務應為選用」[P]。Steinberger 加入 OpenAI 一事僅見二手來源 [S]，repo 內只說 OpenAI 是 donor |
| 規模 / 活動 | GitHub 頁面（WebFetch 摘要）：約 393k stars、82.6k forks、119,589 commits、5k+ open issues、2.8k PRs [P，數字取自摘要]。2026.8.1 release notes：該版 16,977 PRs、987 contributors [P] |
| Release 節奏 | 版本號 `YYYY.M.N`，目前 `2026.10.7`（`package.json`）。docs/releases 顯示 2026 年 10 月至 25 日已有 7 個版本、9 月有 9 個版本，約每週一到數個 [P]。2026.8.1 標示為「OpenClaw 2.0」[P] |
| 名稱沿革 | Warelay（WhatsApp gateway，2025-11-24 前後）→ Clawdbot / Clawd（2025-11-25 起）→ Moltbot（2026-01-27，因 Anthropic 來信商標問題）→ OpenClaw（2026-01-30）。來源：VISION.md 與 docs/start/lore.md [P]。lore.md 也記錄改名當天 X handle 被搶註、Steinberger 個人 GitHub 帳號誤改被搶註、假冒開發者推廣代幣 [P]（其自述帶幽默語氣）。Wikipedia 另列 CLAWDIS（2025-12）[S] |
| 治理注意 | 專案曾於 2026-02-01 啟動 "Stabilisation Mode"（issue #5799）：維護者表示每約 2 分鐘收到一個 PR，無法審查，新 feature 請求改走 skill / 外部工具 / fork；3 月被 VISION.md 取代 [P] |

---

## 2. 定位與使用場景

- 官方定位：「The AI that really does things」，在自己的電腦或主機上跑、透過你已在用的聊天工具互動；個人助理或團隊共用 Gateway「只差設定」[P README/VISION]。
- 使用場景（docs 與 bundled skills 反映）：聊天式日常助理（提醒、筆記 Apple Notes / Obsidian / Bear / Things / Trello / Notion）、Gmail / Calendar / Drive（`gog` skill 包裝 Google Workspace CLI）、email（`himalaya` skill）、瀏覽器操作、智慧家庭（openhue、sonos）、把程式工作委派給 Codex / Claude Code / OpenCode（`coding-agent` skill）、排程與 webhook 自動化、團隊共用 agent（team.openclaw.ai）[P: `skills/` 目錄、docs]。
- 使用者是誰：文件預設是「自己管理 gateway 的 operator」。需要會用終端機、裝 Node、理解 daemon、API key / OAuth、channel 設定（如 Telegram bot token）；Windows / macOS 有桌面 App 降低門檻 [P getting-started]。「非技術使用者」並非主要假設 [推論]：安全文件大量要求使用者自己理解 `sandbox`、`tools.profile`、allowlist。
- 社群 showcase / discussions 未能在此環境檢視，使用者實際用途的量化分布 **未驗證**。

---

## 3. 架構

- **Gateway**：單一長駐 daemon，擁有所有 messaging 連線，預設 bind `127.0.0.1:18789`，提供 typed WebSocket API（JSON Schema 驗證）；每台 host 一個 Gateway [P docs/concepts/architecture.md]。同一 port 也服務 HTTP（Control UI、`/__openclaw__/canvas/`、OpenAI 相容 `/v1/chat/completions`、`/v1/responses`、`/tools/invoke`）[P]。
- **Channels**：WhatsApp（Baileys）、Telegram（grammY）、Slack、Discord、Signal、iMessage、Teams、Matrix、Google Chat、LINE、飛書、WeChat 等 20+，核心只內建 A2A / Reef / Telegram / WebChat，其餘為官方 plugin（`openclaw plugins install @openclaw/<id>`）[P docs/concepts/features.md]。
- **Clients / Nodes**：CLI、TUI、Web Control UI、原生 App 以 WS 連入；macOS / iOS / Android / headless 以 `role: node` 連入，提供 `camera.*`、`screen.record`、`system.run`（遠端執行）等能力，需裝置配對 [P]。
- **Agent runtime**：內建 embedded runtime，並可換 harness（Codex app-server、Claude CLI backend、ACP 外部 coding harness）[P docs/tools/acp-agents.md、README]。支援 multi-agent、sub-agent、`sessions_spawn`、agent-to-agent 訊息（預設開啟）[P trust-model.md]。
- **Sessions / 狀態**：2026.8.1 起 sessions / transcripts 搬進 SQLite；`~/.openclaw/` 下有 `openclaw.json`、`credentials/**`、`state/openclaw.sqlite`、`agents/<id>/agent/openclaw-agent.sqlite`、`workspace/`（AGENTS.md、SOUL.md、USER.md、MEMORY.md、skills/）[P secrets-and-storage.md]。
- **模型接入**：provider plugin 機制，`docs/providers/` 有約 81 頁（Anthropic、OpenAI、Google、Bedrock、Ollama、LM Studio、OpenRouter、LiteLLM 等）；支援 failover 與 per-provider tool policy [P]。

---

## 4. 工具與整合

- **Shell**：`exec` 工具（前景 / 背景 `process`），Linux 上有記憶體上限 watchdog；`apply_patch` 預設啟用且預設限 workspace [P docs/tools/exec.md]。
- **檔案**：`read/write/edit/apply_patch`；`tools.fs.workspaceOnly` 為選用 [P SECURITY.md]。
- **瀏覽器**：OpenClaw 管理的獨立 Chromium profile（與個人瀏覽器隔離），亦可附掛使用者已登入的 Chrome（`user` profile，經 Chrome DevTools MCP）；有 SSRF policy [P docs/tools/browser.md]。
- **Email / Calendar**：Gmail PubSub hook 觸發 agent；`gog`（Gmail / Calendar / Drive / Sheets / Docs）、`himalaya` 等 skill 以 CLI 方式整合 [P]。
- **MCP**：同時是 MCP client（Settings → MCP，Streamable HTTP / SSE / stdio，支援 OAuth）與 MCP server（`openclaw mcp serve`）[P docs/tools/mcp.md]。
- **Plugins**：in-process 載入、視為受信任程式碼，與 OpenClaw 同 OS 權限；VISION 偏好「bundle 型 plugin」（skills + MCP）而非 code plugin [P]。
- **Skills**：目錄內 `SKILL.md`（YAML frontmatter + markdown）；載入優先序為 workspace > project > personal > managed > workshop > bundled；`openclaw skills install/update/verify` 從 **ClawHub**（clawhub.ai）安裝，記錄於 `.clawhub/lock.json`；安裝頁顯示 VirusTotal / ClawScan / 靜態分析結果 [P docs/tools/skills.md]。另有 Skill Workshop：agent 自己寫並更新「learned skills」，每次變更先存舊版、可 undo、30 天未用自動封存 [P]。
- **其他**：Lobster（有核准閘門、可續跑的 workflow runtime）、`show_widget`（在聊天內渲染 HTML widget / 儀表板）、cron、voice call、TTS 等 [P]。

---

## 5. 記憶與主動行為

- **記憶**：純 Markdown 檔，「沒有隱藏狀態」：`USER.md`（偏好，含 observed-date 與 active/superseded 標記）、`MEMORY.md`（長期事實，session 開頭載入）、`memory/YYYY-MM-DD.md`（日誌）、`DREAMS.md`；向量 / 全文搜尋由 memory plugin 提供（同時只能啟用一個 memory plugin）[P docs/concepts/memory.md、VISION]。
- **Dreaming**：背景記憶整併（light / deep / REM 階段），**預設啟用**；`MEMORY.md` 重寫前會保存 preimage [P docs/concepts/dreaming.md]。2026.8.1 起「合格的個人 Claw 可回想該 agent 其他私人對話的內容」[P release notes]。
- **主動行為**：內建 scheduler「Automations」（`openclaw automations`，別名 `cron`）：一次性 / 週期性 job、webhook、Gmail PubSub；輸出可投遞到聊天、webhook 或不投遞。舊的 heartbeat 監控已遷移為一般 automation job（`openclaw doctor --fix` 轉換）[P docs/automation/、docs/gateway/heartbeat.md]。
- **Standing orders**：寫在 `AGENTS.md` 的「長期授權」，定義 scope / triggers / approval gates / escalation rules，讓 agent 在無人提示下按排程執行 [P docs/automation/standing-orders.md]。注意：此授權是 prompt 層，不是程式強制 [推論]。

---

## 6. 威脅模型與權限（最重要）

### 6.1 官方安全立場 [P: SECURITY.md、docs/gateway/security/*]
- 定位為「單一受信任 operator」的個人助理，**不是**多租戶邊界。Gateway 認證通過者＝trusted operator；任何能對 tool-enabled agent 發話的人共享該 agent 的全部授權。
- 模型 / agent **不是**受信任主體（"assume prompt/content injection can manipulate behavior"）；安全邊界來自 host 信任、認證、tool policy、sandbox、exec approvals。
- **純 prompt injection（未跨越上述邊界）、公網暴露、惡意已安裝 plugin、operator 啟用 `dangerous*` 設定，皆被列為「非漏洞 / out of scope」**。這影響我們如何解讀其 CVE 數量：被視為漏洞的只有邊界繞過。

### 6.2 預設權限（關鍵事實）[P]
- Gateway 預設 loopback；DM-capable channel 預設 `pairing`（未知寄件者拿 8 碼配對碼，1 小時過期，每 channel account 最多 3 個待核准；訊息在核准前**不處理**）；群組走 allowlist + mention gate。但**容器映像預設 exposed bind**，部分 workspace channel（如 ClickClack）預設信任成員 [docs/gateway/security/index.md]。
- **Sandbox 預設關閉**（`agents.defaults.sandbox.mode = "off"`）；`tools.exec.host=auto` 在沒有 sandbox 時解析為 **gateway（宿主機）**。
- **Exec 預設無核准**：「No-approval host exec is the default for gateway and node (`mode=full`)」（docs/tools/exec.md）；permission-modes.md 亦寫「Without those settings or sandboxing, the default is full access」。session 可切 `read-only / guarded / workspace(LLM reviewer) / full`。
- 結論：**預設狀態下，能對 agent 發話的人（含被誘導的模型）可以以使用者 OS 權限執行任意 shell**。安全依賴「誰能發話」與使用者自行開啟 sandbox / approvals。docs 內也有矛盾：THREAT-MODEL-ATLAS.md 圖中寫「Docker sandbox (default)」，與 SECURITY.md / exec.md「sandbox off by default」不一致，應以後者（程式與設定文件）為準 [推論]。
- 跨 channel 傳訊：「Agents with message-tool access can send across conversations and channel providers by default」[P]。`tools.sessions.visibility` 預設 `all`，`tools.agentToAgent.enabled` 預設 `true`：未 sandbox 的 agent 可讀所有 agent 的 session transcript [P trust-model.md]。

### 6.3 Credential 儲存 [P secrets-and-storage.md]
- **非加密靜態儲存**：WhatsApp creds、Telegram / Discord / Slack token、model auth profiles（SQLite）、MCP OAuth token 都在 `~/.openclaw/` 內，靠 `700/600` 檔案權限與建議的全碟加密保護；session transcript、tool 參數 / 結果、`.env` 讀取內容「保留原文，包含 credential」。
- 提供 SecretRef（env / file / exec / store provider）與預設關閉的 secret egress proxy（以 sentinel 代替明文，出站時替換）作為強化；`openclaw security audit [--deep] [--fix]` 檢查漂移。

### 6.4 Prompt injection 處理 [P prompt-injection.md]
- 外部內容以 `<<<EXTERNAL_UNTRUSTED_CONTENT>>>` 邊界標記包裹，剝除 ChatML 等 special token，輸出端剝除洩漏的內部標記。
- 官方承認：frontier 模型抵抗力較高（引用 272K 攻擊競技場數據），但適應性攻擊成功率仍可 >80%；建議用 reader agent（唯讀 / 無工具）先摘要不信任內容；弱模型不應用於 tool-enabled agent。
- 結論：**不提供結構性防護**，依賴模型選擇 + 使用者設定。

### 6.5 Lethal trifecta 對應 [推論，基於 P 事實]
- 私人資料：workspace、memory、檔案、email、calendar、跨 session transcript（預設可互讀）。
- 不信任內容：網頁、email、群組訊息、附件、skill 內容、hook payload。
- 外部通訊：message tool 預設可跨 channel 發送、`web_fetch`、`exec`（可 curl）、browser。
- 三者在預設設定下**同時存在於同一個 agent**。官方建議的拆法（reader agent、`tools.profile: messaging`、sandbox、cross-provider messaging 限制）全部是 opt-in。

### 6.6 已驗證的安全事件清單

**A. GitHub Security Advisories [P，github.com/openclaw/openclaw/security/advisories]**
- 規模：分頁約 73 頁（每頁 10 筆）→ 約 730 筆已發布 advisory（由頁數推算，**非明列總數**）；`severity:critical` 共 14 筆（2 頁）；`severity:high` 約 25 頁（約 241–250 筆）。多數 2026-09-11 一批 Moderate / High。注意：量大部分反映維護者把邊界缺陷都公開成 GHSA，並非全是被利用的事件 [推論]。
- 本次逐筆讀取的：

| GHSA | 日期 | 內容 | 嚴重度 | 修補 |
|---|---|---|---|---|
| GHSA-g8p2-7wf7-98mq | 2026-01-31 | Control UI 接受 query 的 `gatewayUrl`，自動連線並把已存 gateway token 送給攻擊者伺服器 → 1-click RCE；即使僅 loopback 也可（瀏覽器發起連線）。npm 套件名 `clawdbot` | High，CVSS 3.1 8.8 | 2026.1.29（需使用者確認新 URL）。GitHub 頁面顯示 "No known CVE"；**CVE-2026-25253 這個編號僅見於二手來源**（Wiz、GitLab Advisory DB、The Hacker News 等）[S]，一手頁面未確認 |
| GHSA-g55j-c2v4-pjcg | 2026-02-04 | 本機行程可無認證呼叫 WebSocket `config.apply`，經 `cliPath` 注入命令 | High，8.4 | 2026.1.20；**CVE-2026-25593**（頁面列出） |
| GHSA-gv46-4xfq-jv58 | 2026-02-14 | `node.invoke`→`system.run` 未剝除內部 `approved` / `approvalDecision` 欄位，已認證 client 可繞過 exec 核准在 node 上執行 | Critical，9.9 | 2026.2.14 |
| GHSA-qrq5-wjgg-rvqw | 2026-02-14 | Plugin 安裝路徑穿越（惡意 package 名） | Critical，9.3 | 2026.2.1 |
| GHSA-4rj2-gpmh-qq5x | 2026-02-14 | voice-call extension 的 inbound allowlist 繞過（空 caller ID + 後綴比對） | Critical | 未讀細節 |
| GHSA-3m3q-x3gj-f79x | 2026-02-14 | voice-call webhook 驗證在特定 proxy 設定下可繞過 | Moderate | 未讀細節 |
| GHSA-rqpp-rjj8-7wv8 | 2026-03-13 | WebSocket shared-auth 連線可自報提升 scope | Critical | 未讀細節 |
| GHSA-4jpw-hj22-2xmc | 2026-03-12 | pairing-scope device token 可鑄造 `operator.admin` 並達 node RCE | Critical | 未讀細節 |
| GHSA-hf68-49fm-59cq / GHSA-fqw4-mph7-2vr8 / GHSA-9hjh-fr4f-gxc4 / GHSA-hc5h-pmr3-3497 | 2026-03-24～29 | 多個 pairing / reconnect / `/pair approve` 路徑把 `operator.read` 或 pairing scope 升級為 `operator.admin`，並達 node RCE | 皆 Critical | 未讀細節 |
| GHSA-8rh7-6779-cjqq | 2026-03-29 | 啟動時先載入 CWD 的 `.env`，惡意 repo 的 `.env` 可覆寫 runtime 安全設定 → config takeover | Critical，9.6 | 2026.3.28 |
| GHSA-j7p2-qcwm-94v4 | 2026-03-29 | 主機 env 黑名單不完整，經 package-manager env 覆寫導向供應鏈 | Critical | 未讀細節 |
| GHSA-9p3r-hh9g-5cmg | 2026-03-31 | remote FS bridge `readFile` TOCTOU 造成 sandbox 逃逸 | Critical | 未讀細節 |
| GHSA-g5cg-8x5w-7jpm | 2026-03-31 | Heartbeat context 繼承經 `senderIsOwner` 升級繞過 sandbox | Critical，CVSS 4.0 9.5 | 2026.3.31 |
| GHSA-xh72-v6v9-mwhc | 2026-04-16 | 飛書 webhook / card-action 驗證改為 fail closed | Critical | 未讀細節 |
| 2026-09-11 批次（如 GHSA-3mq7-q27j-mq7q「Exec approvals 可存活超過其審核的 cwd」、GHSA-9m4p-cqp4-jppq「WhatsApp login tool 可達非 owner turn」、GHSA-rrxp-5mx8-mvhh「來電繼承 owner 工具權限」、GHSA-qgj5-6x35-9g6f「config revision hash 洩漏密碼 verifier」、GHSA-7cp7-87pj-p32v「Skill tool dispatch 略過 owner-only policy」） | 2026-09-11 | 多為 authorization / 邊界一致性缺陷 | High / Moderate | 標題僅自列表取得 |

- 觀察 [推論]：近 3 個月的 Critical 集中在 **scope / pairing / approval / sandbox 的授權邏輯**，而非單點注入——這是「有太多權限邊界要維護」的典型症狀。

**B. 惡意 skill（ClawHub 供應鏈）**
- 一手：repo 內 THREAT-MODEL-ATLAS 將 `T-PERSIST-001`（惡意 skill 安裝）列為 **Critical / P0、residual risk High**，並承認「skill 以 agent 權限執行、無執行期 sandbox」；ClawHub 現行控制為 GitHub 帳號 ≥14 天、靜態 / AST-adjacent 掃描、LLM 風險審查、VirusTotal 掃描（需 operator 設 API key），皆為偵測而非隔離 [P]。2026.8.1 release notes 有「顯示移除惡意 skill 的安全指令」PR [P]。
- 二手 [S]：Koi Security 於 2026-02-01 稽核 2,857 個 ClawHub skill，標示 341 個惡意，其中約 335 個屬同一行動 "ClawHavoc"（假前置步驟，在 macOS 安裝 Atomic Stealer，Windows 用密碼 ZIP）；OpenClaw 隨後整合 VirusTotal；Palo Alto Unit 42 後續仍發現 5 個未被擋下的惡意 skill（The Hacker News、CSO Online、Unit 42）。另有來源稱 1,184 個，與主流數字衝突，視為 **未驗證**。

**C. 暴露實例**
- 二手 [S]：SecurityScorecard 稱 40,214 個暴露實例、12,812 個可 RCE；Hunt.io 稱 17,500+ 個實例受 CVE-2026-25253 影響；Censys / Shodan 數字（21,639 / 42,665 / 37,161）互不一致；DataDome 報告暴露實例被劫持成 scraping botnet。部分文章稱「預設 bind 0.0.0.0」，與一手文件（host 安裝預設 loopback，**僅容器映像預設 exposed bind**）不符，應視為過度簡化 [推論]。fly.md 亦明示 Fly 預設配公網 IP，可被 Shodan / Censys 發現 [P]。

**D. 非產品漏洞但相關**：lore.md 自述的 "Directory Dump"（agent 在群組聊天執行 `find ~` 並分享整個目錄結構）與改名期間詐騙帳號 / 假代幣 [P，自述]。Moltbook（agent 社群網站）資料庫暴露屬另一產品，本報告不採計。

---

## 7. 復原與可稽核性

- **File / action undo**：**未找到**對任意 `exec` / 檔案寫入的通用 undo 或 checkpoint。permission-modes.md 明言「Changing permissions does not undo completed writes or other side effects」。僅有局部機制：workspace 若有 git 則新 workspace 自動 `git init`，文件建議放私有 git repo 備份；Skill Workshop 的變更可 undo；MEMORY.md 重寫有 preimage；`openclaw backup create`（SQLite 安全快照、可排程）與 `doctor` 遷移前備份 [P]。
- **Audit**：Gateway 有 metadata-only 的 audit ledger（誰跑了什麼 agent、工具動作、訊息生命週期、approval decision receipt），**刻意不存 prompt、參數、結果**；另有 per-session trajectory bundle（`/export-trajectory`，prompt、tool calls、模型用量，已 redact）、`openclaw logs`、`agents/tool-policy` 稽核項目、OpenTelemetry / Prometheus [P docs/gateway/audit.md、tools/trajectory.md]。
- Transcript 與 tool 參數／結果保留原文（含 credential），是稽核能力的另一面隱私風險 [P]。
- 判斷 [推論]：「看得到發生什麼」做得相當完整；「撤銷已發生的事」基本沒有。

---

## 8. 模型與成本

- Provider 約 81 個 docs 頁；Claude Code / Codex CLI 既有登入可被 onboarding 重用；本地模型（Ollama、LM Studio、vLLM 等）有專頁 `local-models.md`，並警告本地模型 tool use 與 context budget 問題 [P]。
- 成本顯示：`/status`（token 與估計成本，僅 API key 模型）、`/usage off|tokens|full` 每則回覆 footer、`/usage cost`、`openclaw status --usage`、Control UI Usage 頁（可接 Anthropic / OpenAI Admin API 取得實際花費與每日趨勢）；訂閱制 provider 顯示額度窗口 [P usage-tracking.md]。
- **預算控制**：在 docs 內**未找到**硬性 spend cap / 預算中止機制（`grep budget` 僅命中 context / timeout 預算）；預設單次 agent 執行 timeout 為 48 小時、exec 逾時 1800 秒 [P models.md、exec.md]。「無硬上限」為依我們搜尋的結論，**嚴格說未驗證**。

---

## 9. 上手體驗

- 一行指令：`curl -fsSL https://openclaw.ai/install.sh | bash`（或 `npx openclaw@latest`、`npm i -g openclaw`）。Onboarding 先確立推論：自動偵測 Claude Code / Codex 登入或 API key，做一次真實 completion 驗證，再開 dashboard；其後 `openclaw gateway install` 安裝背景服務 [P getting-started.md]。
- 非技術使用者必須：處理 Node 版本、終端機、取得 API key 或 OAuth、建立 Telegram bot / WhatsApp 配對 / Discord app 等、理解 loopback vs 遠端存取（Tailscale / SSH）、自行打開 sandbox 與 approvals。Quick start 只給一行「安全指南」指標，並不強制選安全層級 [P]。
- 常見失敗點（issues，一手）：升級破壞 [#153257「2026.9.5 把穩定環境變成 8 小時復原」41 則留言]、Gateway 啟動卡 17 分鐘 [#152981]、state lease 衝突 [#159094]、Codex OAuth refresh timeout 導致 cron / heartbeat 失敗 [#89278]、control-ui 在 npm 套件缺檔 [#52808]、bootstrap 檔案被靜默忽略 [#29387]。VISION.md 也把「可靠更新、清楚錯誤、首次設定」列為優先，間接承認這些是痛點 [P]。
- 設定 schema 不保留向後相容別名，靠 `openclaw doctor --fix` 遷移 [P VISION]——對新手是額外負擔 [推論]。

---

## 10. 能否替使用者「做出東西」

- **可以寫 / 跑程式**：agent 擁有 `exec` 與 `apply_patch`，預設在 workspace；可把重度寫程式委派給 Codex / Claude Code / OpenCode（`coding-agent` skill、ACP）；`openclaw` tool 可「委派式設定與修復」自己的設定，並按權限要求核准 [P]。
- **自動化**：cron / webhook / Gmail 觸發 + standing orders；Lobster 提供有審核點的 workflow [P]。
- **「做出小應用」**：`show_widget` 可在聊天內渲染互動 HTML widget 並釘到 dashboard；skill 本身（SKILL.md + 腳本）就是「可維護的小自動化」單位，且 Skill Workshop 讓 agent 自己存 / 修 / 可 undo 地維護 skill [P]。
- **維護性弱點**：沒有「應用 / 自動化」層級的版本控管、測試、回滾、依賴管理；生成的東西散在 workspace 檔案與 skills 內。工程品質取決於使用者是否把 workspace 當 git repo [推論]。對 vibe coder 而言，「建立」支援度高，「安全地維護與還原」支援度低。

---

## 11. 優點 / 缺點 / 使用者痛點

**優點**
1. 通道覆蓋最廣（20+），本地優先，MIT，Foundation 治理，社群與 release 速度極高。
2. 安全文件非常誠實：明列信任模型、非漏洞清單、`security audit` 內建稽核指令。
3. 記憶是可讀 Markdown，使用者可直接編輯 / 版控；`USER.md` 有 supersede 機制。
4. 權限層次細（tool profile、per-agent policy、session permission mode、LLM reviewer）；Skill Workshop 有 undo。
5. Audit ledger、trajectory、OpenTelemetry 等可觀測性完整。

**缺點**
1. **預設不安全的能力組合**：sandbox off、exec 無核准、跨 session 可見、跨 channel 可發訊，安全需 opt-in。
2. 憑證明文落地，transcript 內含 credential 原文。
3. 供應鏈：ClawHub 偵測式而非隔離式，plugin / skill 以完整權限 in-process 執行。
4. 授權邏輯面巨大，持續出現 scope / pairing 升級型 Critical advisory。
5. 沒有通用 undo、沒有硬預算上限。
6. 複雜度極高（docs 數百頁、config key 無數），不適合新手理解全貌。

**使用者痛點（一手 issues）**：升級穩定性（#153257、#152981、#159094）、provider / OAuth 邊角問題（#89278、#38327）、核心過載導致 2026-02 的 Stabilisation Mode（#5799）。成本爆量、token 浪費等常見網路說法，在本次一手資料中**未驗證**。

---

## 12. 對我們的啟發

**值得借鏡**
- 「單一受信任 operator + 明確非目標」的威脅模型寫法，以及把 `security audit` 做成一等指令。
- 記憶用人可讀檔案、帶 provenance 與 supersede；Skill 變更先存舊版可 undo。
- Metadata-only audit ledger 加上可匯出的 trajectory；approval 綁定 exact argv / cwd / env / 檔案 hash（雖然也因此出現 GHSA-3mq7 類漏洞）。
- Reader agent（唯讀、無外部通訊）處理不信任內容的模式。
- Pairing code 預設拒絕陌生寄件者。

**應避免**
- 預設「宿主機全權限 + 無核准」。我們的預設應反過來：sandbox on、唯讀起手、寫入 / 外送需核准、逐步放寬。
- 把授權切成太多 scope（operator.read/write/admin/pairing/approvals）且互相可升級；2026 年 Critical 大多出在此。我們應盡量少角色、單一能力檢查點（capability token）。
- 憑證與 agent 同一檔案系統可讀；應由 OS keychain / 獨立 broker 持有，agent 只拿 sentinel。
- 開放式 skill 市集＋完整權限執行。應採：預設不裝第三方 skill、skill 需宣告權限並在 sandbox 內執行、固定版本 / 簽章。
- 「純 prompt injection 不算漏洞」的立場對研究合理，但對新手產品不可接受；我們需要結構性隔離（資料、不信任內容、外送三者不同時可用）。

**最低安全門檻（由事件史推導）[推論]**
1. 預設 loopback + 認證；遠端存取走受控通道，且 UI 不可被 URL 參數改寫連線目標（GHSA-g8p2 教訓）。
2. 不信任內容 → 不可同時擁有外送能力與私人資料（reader/actor 分離）。
3. 外部副作用（寄信、刪檔、轉帳、發訊息）預設逐次核准，核准內容綁定實際參數。
4. 任何自動執行的路徑（cron、heartbeat、hook）不得繼承比設定者更高的權限（GHSA-g5cg 教訓）。
5. 不從 CWD / workspace 載入會影響安全策略的設定（GHSA-8rh7 教訓）。
6. 檔案變更可還原（git 快照 / copy-on-write），並提供硬性花費上限。
7. 第三方擴充必須隔離執行；供應鏈事件（ClawHavoc）顯示偵測式掃描不足。
8. 執行 `security audit` 類自檢，並在首次設定時展示風險摘要。

---

## 13. 來源與「未能驗證」

**一手來源**
- `https://github.com/openclaw/openclaw`（shallow clone @ `c43d6382`，2026-10-25）：`README.md`、`SECURITY.md`、`VISION.md`、`package.json`、`docs/start/{getting-started,lore,onboarding-overview,why-openclaw}.md`、`docs/gateway/security/{index,trust-model,prompt-injection,secrets-and-storage}.md`、`docs/gateway/{permission-modes,sandbox-vs-tool-policy-vs-elevated,audit,heartbeat}.md`、`docs/tools/{exec,exec-approvals,skills,skill-workshop,mcp,browser,trajectory}.md`、`docs/concepts/{architecture,memory,dreaming,usage-tracking}.md`、`docs/automation/*`、`docs/security/THREAT-MODEL-ATLAS*`、`docs/releases/*`、`docs/install/backups.md`、`skills/*`
- GitHub advisories：`github.com/openclaw/openclaw/security/advisories`（列表頁及 GHSA-g8p2-7wf7-98mq、g55j-c2v4-pjcg、gv46-4xfq-jv58、qrq5-wjgg-rvqw、g5cg-8x5w-7jpm、8rh7-6779-cjqq 個別頁）
- `github.com/openclaw/clawhub` README；issues #5799、#153257、#152981、#159094、#89278、#52808、#29387

**二手來源（僅作線索）**：The Hacker News（341 malicious skills；CVE-2026-25253）、CSO Online、Unit 42、Wiz / GitLab Advisory DB / SentinelOne（CVE-2026-25253）、SecurityScorecard / Hunt.io / DataDome / Infosecurity Magazine（暴露實例）、Wikipedia（名稱沿革、Foundation）

**未能驗證**
- CVE-2026-25253 與 GHSA-g8p2-7wf7-98mq 的對應：一手頁面顯示 "No known CVE"，CVE 編號僅二手可見；NVD 無法連線。
- advisory 總數（約 730）為頁數推算；多數 GHSA 的細節（未標註「已讀」者）僅有標題與日期。
- ClawHavoc 341 / 335 / 1,184 個惡意 skill 數字、所有暴露實例數字、Steinberger 加入 OpenAI 的時間：皆二手。
- openclaw.ai 與 docs.openclaw.ai、Foundation 公告、VirusTotal 合作公告原文：無法連線。
- stars / forks / commits 數字取自 WebFetch 摘要，僅作量級參考。
- 社群 showcase、Discussions 的實際使用分布、成本爆量案例、硬預算機制是否存在：未驗證。
- 沒有實際安裝執行，onboarding 與預設值皆來自文件，未實測。
