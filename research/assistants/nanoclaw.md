# NanoClaw 研究報告

查核日期：2026-10-26。主要來源為官方 repo（github.com/nanocoai/nanoclaw 及其 raw 檔案）。標記「二手」者來自部落格或聚合站，標記「推論」者為本報告作者判斷。

## 1. 基本資料

| 項目 | 內容 | 狀態 |
|---|---|---|
| Repo | `nanocoai/nanoclaw`（搜尋結果與舊文章仍顯示 `qwibitai/nanoclaw`，WebFetch 該 URL 會導向 nanocoai） | 已驗證 |
| 描述 | "A lightweight alternative to OpenClaw that runs in containers for security." | 已驗證 |
| 授權 | MIT | 已驗證 |
| 語言 | Node（host）+ Bun（container 內 agent-runner）+ TypeScript；GitHub 語言比例頁面未顯示 | 部分驗證，來自 README 與 tree |
| 規模 | 31.0k stars、12.7k forks、129 watchers、3,051 commits（main）、287 open issues、737 PR | 已驗證（頁面值，查核當日） |
| 發行節奏 | v2.2.0（8/13）、v2.3.0（8/24）、v2.4.0（9/23）、v2026.10.0（10/9，Latest，首個 calendar-versioned stable）、v2026.10.1-rc.1（10/20）、rc.2（10/22）。約每 1 到 4 週一版，並有 stable/beta/edge 頻道 | 已驗證（releases 頁） |
| 維護者 | README 未點名作者或公司。CONTRIBUTORS.md 列 24 位貢獻者且不標註角色；issue 中可見 collaborator `gabi-simons`、`glifocat`。背後公司（外界稱 Qwibit）只見於二手來源 | 公司未驗證 |
| 與 OpenClaw 關係 | 非 fork、非相容層。README 自述為「同樣核心功能、但小到能讀懂」的替代品，repo topic 含 `openclaw`。沒有找到匯入 OpenClaw 設定或資料的官方工具 | 已驗證（不相容為推論：未見相容宣稱） |

NanoClaw 是活躍的大型專案，並非小型或已棄置的專案。

## 2. 設計目標與取捨

作者動機（README 原話）："I wouldn't have been able to sleep if I had given complex software I didn't understand full access to my life."

相對 OpenClaw，刻意放棄或改變的東西：

- **放棄 application-level 權限檢查，改用 OS-level 隔離**。README 稱 OpenClaw 是「nearly half a million lines of code, 53 config files, and 70+ dependencies」，安全靠 allowlist 與 pairing code。這是作者的說法，我們沒有獨立驗證。
- **放棄設定檔，改為改 code**："No configuration sprawl"，客製化靠修改程式碼，由 Claude Code 協助完成。
- **放棄內建功能**：頻道與其他模型供應商都不在主幹，而是 `/add-<name>` skills，每個使用者的 fork 各自組合。
- **綁定 Claude Agent SDK**（"Best harness, best model"）。Codex、OpenCode、Cursor、Ollama 是可選 provider skill。
- 取捨：原始碼小而容易稽核，代價是每個安裝都是一份獨立 fork，升級與客製化衝突要自己處理（見 issue #3529）。部分 skill 與設定流程依賴 Claude Code，等於預設使用者會使用 coding agent 來維護自己的 agent。

## 3. 架構

- **Host**：單一 Node 行程。負責路由、central DB（agent groups、messaging groups、users/roles、sessions、pending questions）、spawn 與 kill container、投遞回覆。
- **Channels**：WhatsApp、Telegram、Discord、Slack、Teams、iMessage、Matrix、Google Chat、Webex、Linear、GitHub、WeChat、Resend email 等（README 列表）。以 `/add-<channel>` skill 安裝。Adapter 無狀態。
- **Runtime**：每個 session 一個 container（Docker；macOS 可用 Apple Container；Windows 靠 WSL2）。沒有 warm pool，訊息來才 spawn，idle 最長約 30 分鐘後回收（docs/architecture.md，文件自稱 draft，以 code 為準）。
- **Session 狀態**：每 session 兩個 SQLite 檔 `inbound.db`（host 寫）與 `outbound.db`（container 寫），各只有單一 writer，不用 IPC 或 stdin。用 `journal_mode=DELETE` 而非 WAL，因為 WAL 在 VirtioFS 上不可靠。
- **隔離細節**（docs/SECURITY.md）：
  - container 為 non-root、per-session。專案根目錄、host 原始碼與 `.env` 從不掛載。
  - 固定掛載：session 資料夾與 group 資料夾（讀寫），設定、agent-runner 原始碼與 skills（唯讀）。組合後的 `CLAUDE.md` 每次 spawn 重新產生且唯讀。
  - 額外掛載必須符合外部 allowlist `~/.config/nanoclaw/mount-allowlist.json`，沒有該檔就全部禁止。檢查會解析 symlink，並拒絕 `.ssh`、`.env`、`credentials` 等名稱。讀寫需額外 `allowReadWrite`。
  - **網路**：預設對外網路是開放的。可用 `NANOCLAW_EGRESS_LOCKDOWN=true` 讓 agent 進入沒有對外路由的 Docker internal network，只能走 credential gateway；若無法落實則拒絕啟動，不會退回開放。
  - **資源限制**：CPU 與記憶體上限預設未設定（issue #2029 要求可設定 `--memory/--cpus/--pids-limit`）。
- **三種隔離層級**（docs/isolation-model.md）：共用 session；同 agent 不同 session（共用 workspace 與記憶）；不同 agent group（各自 workspace、記憶、container）。

## 4. 威脅模型與權限

- **誰能傳訊息**：存取需要 `agent_group_members` 成員身分；管理指令需要 owner/admin（`user_roles`）。未註冊寄件者依各 messaging group 的 `unknown_sender_policy` 處理。所有訊息，包含成員的，一律視為可能的 prompt injection。
- **憑證處理**：真實 API 憑證不進入 container（不在 env、檔案、stdin、`/proc`）。出站 HTTPS 經 credential gateway 在網路邊界注入。NanoClaw 本身不內附 gateway，必須透過 skill 安裝（預設 OneCLI，另有 Iron Proxy），**沒有 gateway 時 host 拒絕啟動**，閘道掛掉時會拒絕待決請求、停止執行中的 session。
- **預設核准行為**：agent 的 container 內操作（bash、檔案）不需逐步核准，因為隔離就是邊界。需要憑證的外部動作由 gateway 產生 policy hold，核准卡片送到 approver 的 DM（scoped admin、global admin、owner 的順序），一次卡片只核准一個請求；所有失敗路徑均為 deny。`install_packages`、`add_mcp_server`、`create_agent` 這類自我修改動作經由 host 的 system action，通常需要 admin 核准。
- **供應鏈**：pnpm 設 3 天最小 release age、install script 需人工核准。
- **官方承認的限制**（SECURITY.md「Known risks」）：egress 預設開放，非 proxy-aware 的流量可繞過 gateway、核准與稽核；prompt injection 仍是風險；無預設資源上限；安全依賴另外安裝的 gateway；symlink race 只在支援 anchored-directory 的檔案系統完全處理。
- **推論**：agent 的 group 資料夾（含 skills 與 CLAUDE.md）可讀寫，agent 可改寫自己的指令與工具，被注入後可能持久化惡意指令；這屬於 container 外殼擋不住的層面。issue #411 要求補充 indirect prompt injection 說明（open，good first issue）。issue #1149 是 2026-03 的一個 priority/critical 安全回報，仍為 open 且無公開細節。

## 5. 工具、記憶與主動行為

- **工具**：agent 內建 messaging 工具（`send_message`、`send_file`、`send_card`、`ask_user_question` 等）與 Claude Agent SDK 的 bash/檔案工具。自我修改：`install_packages`、`add_mcp_server`、`create_agent`。
- **Skills**：capability skill（加頻道、供應商、工具）與 patch skill（小修）。安裝是「複製檔案」而非 git merge，要求可重複執行並附 remove 步驟。公開 registry 的 skill 需審查，每個新版本重審。
- **記憶**：純 Markdown 檔在 `groups/<folder>/memory/`，採 Open Knowledge Format v0.1，無資料庫與 embedding。新 context window 啟動時 hook 注入 `index.md` 與 `system/definition.md`（各上限 16k 字元）。使用者可直接用編輯器改。
- **排程**：`ncl tasks`（一次性 `--process-after` 或 cron `--recurrence`）。每個任務跑在自己的 system session，不綁聊天，prompt 必須指名輸出目的地。未加 script gate 的週期任務若一天觸發超過 4 次會被拒絕；script gate（30 秒、1 MB 輸出）可回傳 `{"wakeAgent": false}` 以省 token；連續失敗做指數退避，8 次後暫停。範本任務建立時為暫停狀態，不會未經同意啟動背景工作。

## 6. 復原與可稽核性

- 升級走 `/update-nanoclaw`，會先 snapshot，build 或 health check 失敗則還原；migration 只往前。`data/upgrade-state.json` 記錄版本、commit、tree，host 啟動時比對，不符就拒絕啟動（防止用 raw `git pull` 繞過）。
- docs/upgrade-recovery.md **沒有描述 rollback 程序**；hardened image 有 opt-out 路徑。
- 稽核：每個 session 的 DB 與 `conversations/` 保留訊息歷史，setup 有 `logs/setup.log`。文件未見獨立的「誰在何時核准了什麼」審計日誌（未驗證，僅確認所讀的文件未提及）。核准卡片與 `pending_questions` 存在 central DB。
- 已知穩定性問題：#3716 PreCompact 封存每次完整重寫檔案，被回報造成 OOM crash loop；#2312 `groups/global/CLAUDE.md` 每次啟動被刪除。

## 7. 模型與成本

- 主要為 Claude（Agent SDK）；v2.4.0 release note 稱預設 Claude 模型改為 Opus 5.5，並支援 Codex、OpenCode、Cursor、Ollama（本地、plain-HTTP 經 Iron Proxy）。每個 agent group 可設定不同 backend。
- 認證可透過 `claude setup-token`（訂閱 OAuth）或 API key。issue #1608 指出 OAuth 設定「confusing and undocumented」。
- 成本控管：找不到內建的預算或花費上限（未驗證，僅指所讀文件）。排程任務每日 4 次的限制與 script gate 屬於間接節流。
- 隱私：README 稱沒有使用者帳號，匿名 setup 診斷可用 `NANOCLAW_NO_DIAGNOSTICS=1` 關閉；預建 hardened image（約 800 MB）需要免費 NanoClaw 帳號，收集 email、IP（/24 截斷）等。

## 8. 上手體驗（非技術使用者）

- 入口：clone 後執行 `bash nanoclaw.sh`，腳本自動安裝 Node、pnpm、Docker（若缺），建 container 映像，交接 Claude 登入，詢問顯示名稱與頻道，並做 Telegram 等頻道配對。官方示例約 4 分 42 秒，最久的是 container build。步驟失敗時腳本會叫 Claude Code 診斷並續跑。
- 需求：macOS/Linux（Windows 要 WSL2）、Docker Desktop 或 Engine、Claude Code（`/customize`、`/debug` 與所有 `/add-*` skill 都需要）、Anthropic 憑證。
- **推論**：對沒碰過終端機與 Docker 的 vibe coder 仍有門檻：要會開終端機、裝 Docker Desktop（背景常駐、吃記憶體）、理解 OneCLI gateway。Claude Code 輔助降低了錯誤排除難度，但等於上手流程依賴另一個 agent。
- 已知痛點 issue：#1184（受限 K8s 環境部署困難）、#1485（想把 NanoClaw 本身放進 container）、#1503（官網 SSL 憑證失效，19 則留言）。

## 9. 能否替使用者做出東西

可以。container 內是完整的 Claude Agent SDK，有 bash、檔案編輯、Chromium、Node/Bun/git，agent 能寫並跑程式，把成果放在自己的 group workspace，也能 `install_packages`（需核准）。小型自動化可以用「排程任務 + script gate」完成。限制：agent 只能碰掛載進來的東西；要動使用者主機上的專案，需要使用者把目錄加入 mount allowlist；要呼叫外部服務，憑證需經 gateway 並經過核准卡片。它是「在隔離環境裡替你做事」的模型，不是「在你的專案資料夾裡直接改」的模型。

## 10. 優點 / 缺點 / 痛點

優點：
- 隔離為預設，且是 OS 級；憑證不進 container；gateway 不在就不啟動，失敗全部 deny（fail closed）。
- 文件誠實列出已知缺口（egress 預設開放、無資源上限）。
- 活躍度高，release 流程成熟（頻道、RC、升級狀態檢查）。
- 記憶是人類可讀的 Markdown，可手動編輯。

缺點與痛點：
- 安全預設不完整：egress lockdown、資源上限預設關閉，要使用者主動開。
- 強依賴 Docker、Claude Code 與外部 gateway；每個安裝是一份 fork，升級衝突見 #3529。
- 無 warm pool，每次喚醒有冷啟動成本（推論：延遲幅度未量測）。
- 核准卡片與 gateway 設定對初學者是新概念。
- 287 個 open issues 與 737 個 open PR，維護頻寬是風險（推論）。

## 11. 對我們的啟發

**隔離預設買到什麼**
- 買到：(1) agent 執行 bash 的爆炸半徑限於掛載目錄，host 的 `.ssh`、`.env`、其他專案不會被碰到；(2) 憑證不在 agent 可讀範圍，被注入也偷不走金鑰（只能「借用」經 gateway 的授權呼叫，而那需要人核准）；(3) 失控或當機的行程可整個丟掉重建；(4) 不同聊天室/使用者的 agent 彼此看不到資料。
- 沒買到：prompt injection 本身（agent 讀到惡意內容後，仍可使用它「被允許」的一切：寫入自己的指令檔、透過已授權頻道外洩 workspace 內的資料、在 egress 開放時直接打外網）。egress 預設開放使「憑證不外洩」之外的資料外洩防護很弱。這是官方文件自己承認的。
- 結論（推論）：isolation-by-default 把「災難性的 host 破壞與憑證竊取」降到很低，但「被騙去做允許範圍內的壞事」仍需靠 egress allowlist、核准與稽核。真正的安全來自 container + egress 預設鎖 + 憑證代理 + 人類核准四件組合，缺一就打折。

**UX 代價**
- 安裝重（Docker、約數分鐘 build、約 800 MB 映像）、冷啟動、WSL2/VirtioFS 之類跨平台怪問題、客戶端需要 Claude Code。
- 「agent 看不到我的專案」是雙面刃：安全，但使用者要學會「掛載」概念，才能讓它改自己的專案。
- 核准卡片頻率若設計不當會造成核准疲勞。NanoClaw 把 container 內動作免核准、只對憑證出站核准，是一個值得借鑑的折衷。

**可借鑑**：host 協調 + 單寫入者 SQLite mailbox；「failure 一律 deny」；沒 gateway 不啟動；範本任務預設暫停；升級狀態標記；skills 以「複製 + 可移除」而非 merge。
**可避免**：預設 egress 開放；預設無資源限制；對初學者要求自行維護 fork。

## 12. 來源與「未能驗證」

主要來源（官方）：
- https://github.com/nanocoai/nanoclaw（主頁、tree、releases、issues）
- https://raw.githubusercontent.com/nanocoai/nanoclaw/main/README.md
- 同路徑 `SECURITY.md`、`docs/SECURITY.md`、`docs/architecture.md`、`docs/isolation-model.md`、`docs/memory.md`、`docs/scheduled-tasks.md`、`docs/skills-model.md`、`docs/gateway-seam.md`、`docs/hardened-image.md`、`docs/setup-flow.md`、`docs/upgrade-recovery.md`、`CONTRIBUTORS.md`
- Issues：#1149、#411、#1608、#1184、#1485、#1503、#2029、#2312、#3529、#3716

二手（僅用於交叉比對）：WebSearch 結果中的 trendingtopics.eu、screenshotone.com 文章、聚合站對 stars 與行數的不一致說法（4,000 行、17,500 行、2,000 行），因與官方不符而未採用。

未能驗證：
- GitHub API 無法存取（403），所以無首次 commit 日期、最後 commit 日期、語言比例。頁面也未顯示這些。
- 背後公司（Qwibit）與維護者身分。
- 程式碼行數與「容易稽核」的實際程度（未跑 cloc）。
- 與 OpenClaw 的資料相容或遷移工具。
- 實際冷啟動延遲、真實使用的 token 成本。
- 這些文件描述的行為是否與目前 code 一致（docs 自稱部分為 draft，以 `buildMounts` 為準；我們沒有讀 source）。
- docs.nanoclaw.dev 的 canonical 文件未讀。
