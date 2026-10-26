# ZeroClaw 研究報告

查核日期：2026-10-26。主要來源為官方 repo（github.com/zeroclaw-labs/zeroclaw 與 docs/book/src 原始檔）。標記「二手」為第三方頁面，「推論」為本報告作者判斷。

## 1. 基本資料

| 項目 | 內容 | 狀態 |
|---|---|---|
| Repo | `zeroclaw-labs/zeroclaw`。README 聲明這是唯一官方 repo，並警告其他同名 repo、package、網域與官方無關（搜尋結果可見 rareba、enzomar、creke 等大量 fork/鏡像） | 已驗證 |
| 描述 | "Fast, small, and fully autonomous AI personal assistant infrastructure, any OS, any platform" | 已驗證 |
| 授權 | MIT 或 Apache-2.0 雙授權 | 已驗證 |
| 語言 | Rust（Edition 2024 badge；GitHub 語言比例頁面未顯示） | 已驗證（依 badge） |
| 規模 | 33.0k stars、5.0k forks、81 watchers、5,760 commits（master）、646 open issues、412 open PR | 已驗證（頁面值） |
| 發行節奏 | v0.8.3（7/16）、v0.8.4（8/2）、v0.8.5（9/5）、v0.8.6（10/18，Latest）。約每 2 到 5 週一版，release notes 稱 v0.8.6 含 573 commits、74 contributors。仍是 0.x，有 breaking changes（v0.8.4 改了 skill sources、Nextcloud Talk、webhook 認證） | 已驗證 |
| 維護者 | 治理文件（FND-003）描述為「lightweight, student-led team」，單一 GitHub team `core-contributors`，不具名，未提公司或基金會。有 CLA | 已驗證；個人身分與商業實體未驗證 |
| 與 OpenClaw 關係 | 獨立重寫，非 fork。官方 README 與 philosophy 頁**完全不提 OpenClaw**，只有 repo topic 標 `openclaw`。第三方鏡像（v0.1.8 時期文件）顯示曾有 `zeroclaw migrate openclaw --dry-run` 匯入 `~/.openclaw/` 設定與記憶的功能；目前 master 的 CLI reference 路徑 404，無法確認現行版本是否仍有 | 重寫為推論；migrate 指令僅二手，現況未驗證 |

這是活躍的大型專案。注意 README 的 "30+ channels"、效能數字（RAM < 5 MB、約 8.8 MB binary）是專案自述，官方主頁並未附方法，不在本報告驗證範圍。

## 2. 設計目標與取捨

官方 Philosophy 有四條，依優先順序：You own it；Security-first, with escape hatches；Minimal（binary 大小、依賴、表面積）；Provider-agnostic。「What this isn't」：不是 SaaS、不只是聊天 UI、不是 framework（你設定它而不是在上面寫 app）、不是玩具（目標是 24/7 在 homelab/VPS 上運行，可註冊為 systemd/launchd/Windows Service）。

相對 OpenClaw（推論，因官方未比較）：以 Rust 單一 binary 取代 Node runtime；以 TOML 單檔設定（`~/.zeroclaw/config.toml`）；提供 trait 化的 provider/channel/tool/memory 擴充點；加入 WASM 外掛（Extism / wasmtime）；加入 SOP engine 與 hardware peripheral。與 NanoClaw 相反，它**沒有走「少功能、可讀完」路線**：5,760 commits、30 多個 channel、硬體、ACP、OIDC、A2A 等，功能面其實很大。它的「minimal」是指 runtime 資源與依賴，不是功能範圍。

## 3. 架構

- **進入點**：channels、HTTP/WebSocket gateway（附 web dashboard，預設 `http://127.0.0.1:42617`）、ACP（Agent Client Protocol）。往下是 runtime（agent loop、security policy layer、SOP engine），再往下是 providers、tools、memory。
- **Channels**：Telegram、Discord、Slack、Matrix、Signal、WhatsApp、LINE、Mattermost、Email、語音、webhook、MQTT、AMQP、Git forge 等（SUMMARY.md 目錄）。
- **隔離機制**（docs/security/sandboxing.md，這是最關鍵的細節）：
  - 這是「OS 級 sandbox 包住每次 tool 執行」，不是整個 agent 住在 container。後端：Landlock（Linux 5.13+）、Firejail、Bubblewrap、macOS Seatbelt（`sandbox-exec`）、Docker（每次呼叫一個暫時 container，100 到 500 ms 啟動成本）、none。
  - `sandbox_backend = "auto"` 的偵測：Linux 先 Landlock（僅編進去才有）再 Firejail；macOS 先 Bubblewrap（僅編進去才有）再 Seatbelt；**Windows 沒有 OS 級後端**。Landlock 與 Bubblewrap 是 compile-time feature，**不在預設與 release feature 集合**，所以一般 release build 在 Linux 用 Firejail，macOS 用 Seatbelt。
  - 偵測不到任何後端時，native runtime 靜默落到 `none`，只剩 application-layer 檢查（log 會寫 "No sandbox backend available, using application-layer security."）。
  - **掛載/檔案**：可讀 workspace、`/usr`、`/lib`、`/etc`（唯讀）；可寫 workspace 與 `/tmp`；`forbidden_paths`（預設含 `/etc`、`/sys`、`/boot`、`~/.ssh`）。環境變數只傳 `shell_env_passthrough` 列出的。
  - **網路**因後端而異：Landlock 與 Firejail 不變（即不限制）；Bubblewrap 無網路；Seatbelt 阻擋出站（DNS 與 localhost 除外）；Docker sandbox `--network none`。tool 層另有 `allowed_domains`，`http_request` 預設擋私有與本機目標。
- **狀態儲存**：cron 在 `data/cron/jobs.db`；成本在 `<workspace>/state/costs.jsonl`；委派任務在 `data/control_plane.db`；記憶後端為 SQLite、Postgres、Lucid、Qdrant、Markdown 等，預設後端文件未明言。

## 4. 威脅模型與權限

- **自主等級**（docs/security/autonomy.md）：`readonly`、`supervised`（**預設**）、`full`。supervised 之下：低風險（`file_read`、`http GET`、`memory_search`）自動執行；中風險（workspace 內 `file_write`、白名單內 shell 指令、對允許網域 `http POST`）需操作者核准；高風險（未知或被拒的 shell、workspace 外寫入、破壞性模式）直接封鎖。
- **核准方式**：CLI 內嵌提問、Telegram inline keyboard、Slack Block Kit 按鈕；Discord/Signal/Matrix/WhatsApp 則回覆「token approve|deny|always」。逾時（預設 120 秒）算拒絕；`on_no_approver` 預設 deny（fail closed）。可用 `approval_route` 把核准送到獨立的 approver 頻道。
- **誰能傳訊息**：Telegram 範例：沒有任何 peer 時產生一次性配對碼，使用者在 Telegram 輸入 `/bind <code>`，ID 寫入 config；對陌生人，bot 回覆一條建議給操作者的指令。`external_peers = ["*"]` 會開放所有人。Gateway 端為 bearer token 配對，遠端沒憑證直接拒絕；principal 權限 deny-by-default。**注意：頻道身分尚未對映到 principal 模型**（authentication.md 原文），#5982 要求 per-sender RBAC，仍是 open。
- **憑證處理**：sandbox 移除對父行程 secrets 的存取；leak detector 掃**出站**頻道回覆並遮蔽（固定樣式加可選的高熵啟發式）；此偵測不掃入站內容。
- **Prompt injection 緩解**：模型輸出經 injection guard 掃描已知樣式；shell validator 檢查危險旗標與管線；allow/deny list；`excluded_tools` 可對特定頻道隱藏工具；可選 OTP。官方明說這些是 pattern-based，新型攻擊可能通過。
- **官方承認的限制**（security/model.md）：命令執行**不被稽核**（`[security.audit]` 只涵蓋憑證簽發與更新）；tool receipts 是弱證據；沙盒覆蓋因平台而異，Windows 無；`full` 模式削弱閘門；預設 OTP 與緊急停止皆關閉。
- **推論**：README 的「Every action gets cryptographic tool receipts」與「OS-level sandboxes」措辭強於實作文件（receipts 金鑰只在記憶體、無持久儲存、不鏈接；一般 build 沒有 Landlock/Bwrap）。評估時應以 docs/security 為準而非 README 摘要。
- **YOLO 模式**：一鍵關閉核准、workspace 邊界、shell policy、forbidden paths、sandbox、OTP、estop、gateway pairing。官方警告勿用於共用主機或對外公開的 bot。

## 5. 工具、記憶與主動行為

- **工具**：預設 registry 很大：`shell`、`file_write/edit`、`git_operations`、`http_request`、`browser_open`、`web_search`、`cron_*`、`spawn_subagent`、`delegate`、`backup`、`model_routing_config`、`proxy_config` 等。可選的 `claude_code`、`codex_cli`、`gemini_cli` runner、Notion/Jira/Google Workspace/Microsoft365 整合。官方清單中，風險工具與預設啟用工具重疊多，所以靠 autonomy 閘門而非「預設不載入」。
- **Skills**：`SKILL.md` 或 `SKILL.toml`，來源可為本機、Git、registry、well-known HTTPS 索引（須指定 `--skill`）、簽章 bundle。載入前有安全審核；腳本檔（.sh 等）預設封鎖，需 `skills.allow_scripts` 才能開。**Agent 自動建立 skill 預設關閉**，開啟後可把成功的多步驟任務存成 skill。
- **排程與主動行為**：cron job 記錄擁有它的 agent alias，用該 agent 的安全政策執行；文件**未描述 cron 建立是否需要核准**（未驗證）。SOP engine 可由 MQTT、webhook、cron、peripheral 事件觸發，具核准閘門與可恢復的 run（`sop.persist_runs` 才存 SQLite，預設在記憶體）。背景 delegate 重啟後標記為 lost，不會續跑。
- **記憶**：依 ADR-010 有記憶權限邊界；工具 `memory_store/recall/forget`；dashboard 可瀏覽。embedding 本地或遠端文件未說明（未驗證）。

## 6. 復原與可稽核性

- 日誌：JSONL，可輸出 OTLP；預設滾動 200 事件窗口；工具輸入輸出預設遮蔽，LLM payload 預設不記。官方明確說不是 transactional audit guarantee，事件可能遺失。
- Tool receipts：HMAC-SHA256，用來防止模型捏造「我跑過這個工具」，金鑰只在記憶體，不持久、不鏈結，不能當稽核紀錄。
- 核准、授予、拒絕、逾時會發出結構化事件。
- 成本帳本是 append-only 的 `costs.jsonl`。
- 回復：observability 與 security 文件找不到 rollback 或 undo 機制；`zeroclaw estop` 在 YOLO 下失效。`backup` 工具存在但細節未讀。升級有 `zeroclaw config migrate`。整體來說可稽核性弱於 NanoClaw 的「容器邊界 + 核准紀錄」嗎？推論：兩者都沒有完整的操作稽核；ZeroClaw 明說命令執行不稽核。

## 7. 模型與成本

- Provider 無關：Anthropic、OpenAI、Ollama 與約 20 個其他，任何 OpenAI-compatible 端點，另有 fallback 鏈與路由、OpenAI Codex 訂閱登入、Claude token（`claude setup-token`）。
- 成本追蹤（ops/cost-tracking.md）：記錄每次呼叫，按 agent 歸屬，每日/每月預算；`enforcement.mode` 預設 **`warn`**（只警告，不擋），可設 `block` 或 `route_down`（換便宜模型）。只計有定價的花費，未定價 token 不計入，`zeroclaw status` 會提醒。**預設不會因超支而停止**。
- 本地小模型：issue #5287 在討論 `local_small` 執行設定檔，代表目前小模型體驗仍在改進。

## 8. 上手體驗（非技術使用者）

- 安裝：`curl ... install.sh | sh` 後 `zeroclaw quickstart`；Windows 有免 Rust 的預建路徑；Homebrew、cargo 亦可。單一 binary，不需 Docker 或 Node，這比 NanoClaw 輕。
- Quickstart 逐項詢問：provider 與認證、模型、agent alias、risk profile；頻道可跳過（CLI channel 立即可用）。三種介面：CLI、zerocode 終端 TUI、網頁 `/quickstart`（需 daemon 在跑）。文件未給時間估計。
- 對初學者的雷：要懂 provider、risk profile、alias 這些概念；設定寫完要手動 `zeroclaw service restart`；plugin 步驟涉及簽章與 egress 授權；Telegram 配對碼印在前景輸出或日誌，要去終端機抄；config 太舊需 `config migrate`；Windows 沒有 OS sandbox，且 issue #7462 回報 74 個 Windows 測試失敗。
- **推論**：文件為 mdBook，內容龐大（60 多個頁面目錄），對沒有 CLI 習慣的人偏重。

## 9. 能否替使用者做出東西

能，但路徑不同：agent 直接在使用者機器的 workspace 內執行 shell、寫檔、git，甚至呼叫 `claude_code`/`codex_cli` 這些 coding CLI 當子工具。小自動化可靠 cron、SOP、skill（agent 可把成功流程存成 skill，預設關閉）。supervised 預設下，每次 file_write 與 shell 都要按核准，所以「做一個小 app」的流程會產生大量核准提示，除非設 `auto_approve` 或切到 `full`/YOLO，而那會拆掉安全邊界。這是 ZeroClaw 最大的 UX 與安全拉扯。

## 10. 優點 / 缺點 / 痛點

優點：
- 單一 binary、無 Docker 依賴、低資源（專案自述，未獨立驗證）。
- 安全政策分層明確：autonomy、allowlist、forbidden paths、OS sandbox、OTP、核准路由，且失敗預設 deny；SECURITY.md 有 48 小時回應 SLA；有 SLSA provenance 文件。
- 文件自我揭露限制（命令不稽核、receipts 弱、Windows 無沙盒），這類誠實度值得肯定。
- 成本帳本、provider fallback、`route_down` 對成本控制友善。

缺點與痛點：
- 隔離不是預設的 container 級：一般 build 的 sandbox 只是 Firejail/Seatbelt，Windows 沒有，Firejail 與 Landlock 不限制網路；偵測不到就靜默降級到應用層檢查。
- 功能面很大（硬體、30+ channel、外掛），攻擊面與維護面大；0.x 版有 breaking changes；646 open issues、412 open PR。
- 高風險 open issues：#9328（verifiable-intent 未驗證憑證鏈，risk:high）、#6996（要求更細緻的 sandbox 政策）、#8424（forbidden path 模式）、#5982（per-sender RBAC）。
- 無稽核紀錄、無回復機制；成本預設只警告。
- 供應來源混淆：大量同名 fork 與冒名網站，官方自己發出 impersonation 警告。

## 11. 對我們的啟發

**ZeroClaw 的觀點：隔離做在「每次工具呼叫」而非「整個 agent」**。優點是輕、快、單檔；缺點是安全品質取決於平台後端，最差情況退到純 pattern 檢查。對我們（面向初學者、safety-first）的意義：

- isolation-by-default 買到的是「即使 agent 被騙，也讀不到 `~/.ssh` 或寫不出 workspace」。ZeroClaw 的 Firejail/Seatbelt 路線能買到大部分檔案邊界，但**網路**常常沒被限制，意味資料外洩防護弱。NanoClaw 的容器路線檔案與憑證邊界更強，但 egress 預設也開放。兩者共同教訓：**檔案隔離容易，網路出口與憑證代理才是難點**，產品預設應該同時處理。
- UX 代價：NanoClaw 付的是安裝重量（Docker、WSL2、冷啟動）與「看不到我的專案」；ZeroClaw 付的是核准疲勞（supervised 對每個中風險動作都問）與設定概念複雜（risk profile、alias、peer group）。對 vibe coder，兩種代價都會導致使用者最後關掉安全設定（ZeroClaw 直接提供 YOLO；NanoClaw 以 gateway 與 allowlist 不友善換取安全）。
- 可借鑑：fail-closed 預設；風險三級（低自動、中問、高擋）；核准送到獨立管道；逾時視為拒絕；成本帳本；sandbox 偵測失敗時要**大聲失敗**而非靜默降級（ZeroClaw 這點做得不好）；配對碼流程；agent 自建 skill 預設關閉。
- 應避免：README 行銷措辭強於實作（receipts、OS-level sandbox）；預設 `warn` 的成本控管；隱藏的 compile-time sandbox feature；工具預設全開再靠閘門擋。

## 12. 來源與「未能驗證」

主要來源（官方）：
- https://github.com/zeroclaw-labs/zeroclaw（主頁、docs 樹、releases、issues）
- https://raw.githubusercontent.com/zeroclaw-labs/zeroclaw/master/README.md 與 `SECURITY.md`
- docs/book/src 下：`SUMMARY.md`、`philosophy/index.md`、`philosophy/what-this-isnt.md`、`security/model.md`、`security/sandboxing.md`、`security/autonomy.md`、`security/authentication.md`、`security/tool-receipts.md`、`getting-started/quickstart.md`、`getting-started/yolo.md`、`channels/telegram.md`、`tools/skills.md`、`developing/tool-inventory.md`、`architecture/background-work-lifecycle.md`、`architecture/memory-payload-lifecycle.md`、`ops/cost-tracking.md`、`ops/observability.md`、`foundations/fnd-003-governance.md`
- Issues：#7462、#9328、#6996、#8424、#5982、#5287

二手：pinggy.io 部落格、jimmysong.io 聚合頁、git.polymech.info 對舊版（v0.1.8）文件的鏡像（用於 OpenClaw migrate 指令）、kilo.ai 比較頁（搜尋摘要可見，但頁面抓取失敗，未採用內容）。

未能驗證：
- GitHub API 403，無首次 commit 日期、最後 commit 日期、貢獻者總數、語言比例。
- 目前版本是否仍有 `zeroclaw migrate openclaw`（現行 CLI reference 404）。
- README 的 RAM < 5 MB、binary 約 8.8 MB、"30+ channels" 等數字（專案自述）。
- 幕後團隊的具體身分與是否有商業實體。
- cron 建立是否需核准；預設記憶後端與 embedding 是否本地；`backup` 與 rollback 實際能力。
- 一般 release build 實際編入哪些 sandbox feature（以 docs 描述為準，未檢查 Cargo.toml 或 release artifact）。
- 實際上手時間與核准提示頻率（未實測）。
