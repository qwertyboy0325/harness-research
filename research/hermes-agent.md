# Hermes Agent 研究報告

查證日期：2026-10-26。方法：以 github.com 頁面（repo 首頁、releases、issues 列表、目錄樹）與 raw.githubusercontent.com 上的 repo 內文件原始檔（README.md、AGENTS.md、website/docs/**）為主要來源。官方文件站 hermes-agent.nousresearch.com 本次未嘗試直接抓取，文件內容讀的是 repo 內 Docusaurus markdown 原始檔，並非渲染後網站。未讀 Python 原始碼本體，實作細節處標註「文件所述」。WebFetch 回傳為小模型摘要，非逐字原文，數字請以原始 repo 為準。

## 1. 基本資料
- Canonical repo（verified）：**github.com/NousResearch/hermes-agent**，About 描述 "The agent that grows with you."；README 稱 "The self-improving AI agent built by Nous Research"。與 Nous 的 Hermes LLM 系列是不同產品（README 宣稱 Nous Portal 提供 300+ 模型）。
- License：MIT（verified，repo 頁面）。
- 語言/技術棧：核心為 Python（run_agent.py、cli.py、hermes_state.py 等；AGENTS.md verified）；TUI 為 Ink（Node，ui-tui/）搭配 Python JSON-RPC 後端（tui_gateway/）；桌面 App 為 Electron（apps/desktop/）；dashboard SPA（web/）；文件站 Docusaurus（website/）。GitHub 語言比例本次頁面未顯示（unverified）。環境管理用自家 `pm/`（uv 系）。
- 組織：Nous Research。資金/商業模式：README 與 docs 顯示有 **Nous Portal** 訂閱（300+ 模型 + Tool Gateway：web search、FAL 圖像、OpenAI TTS、Browser Use 雲端瀏覽器；`hermes setup --portal` OAuth）。Nous 的融資情況本次未查（unverified）。商業模式「軟體 MIT 開源、以 Portal 訂閱導流變現」是依文件的推論。
- 活躍度（repo 頁面，2026-10-26）：257.1k stars、56.3k forks、987 watchers、main 上 59,182 commits、open issues 與 open PR 皆顯示 "5k+"（無確切數字）。規模極大，與 secondary 來源的星數不一致，此處只採頁面數字。
- Release：releases 頁最新為 **v0.22.5（2026-10-22）**，其前 v0.22.4（10-21）、v0.22.3（10-21，約 620 merged PRs）、v0.22.2（10-15，取代 build 錯 commit 的 v0.22.1）、v0.22.1（10-14）。即約每週多個版本，仍在 0.x。repo 首頁的 Releases 區塊本次顯示為空，releases 頁才有資料（兩者不一致，原因 unverified）。

## 2. 架構
（AGENTS.md、desktop.md、sessions.md，verified）
- 單一 agent core 供 CLI、messaging gateway（AGENTS.md 稱約 20 平台）、TUI、Electron 桌面 App 共用。Desktop 文件稱與 CLI/gateway 是「同一個 agent」，共享 config、API keys、sessions、skills、memory；Desktop 對話跑在 gateway 上，可跨介面 resume，本機使用不需另起 `hermes serve`。
- Agent loop：`run_agent.py` 的 `AIAgent` facade，回合邏輯在 `agent/turn_*.py`（各階段細節文件未述）。工具由 `tools/registry.py` 註冊、`model_tools.py` 發現與分派、`toolsets.py` 分組。
- 設計不變量：per-conversation prompt prefix 固定以保 prompt cache；中途改 toolset/memory/system prompt 會破壞 cache，唯一例外是 compression。「核心要窄」：每個 tool 每次 API call 都送出，新能力優先走 CLI+skill、service-gated tool、plugin，最後才新增 core tool。
- 狀態儲存：所有 session（CLI 與 messaging）存 SQLite `~/.hermes/state.db`，含訊息、metadata、token 數，FTS5 全文檢索（sessions.md verified）。設定 `~/.hermes/config.yaml`，祕密 `~/.hermes/.env`，記憶 `~/.hermes/memories/`，skills `~/.hermes/skills/`，checkpoints `~/.hermes/checkpoints/`。支援 profiles（多 profile、單一 gateway 服務多 profile）。
- 重構痕跡：AGENTS.md 提到「former god files 改為 facade + sibling」，open issue #78647「Repo-wide godfile eradication」有 82 則留言，顯示程式碼仍在大幅整理。

## 3. 工具集
- 「40+ tools」：README 寫 "40+ tools"；toolsets-reference.md 實際列 **71 個 unique built-in tools**（不含 MCP/plugin/context_engine）。結論：宣稱偏保守，但其中含大量平台專屬工具（feishu、yuanbao、discord_admin、kanban 14 個、desktop_ui 11 個），一般 coding 會用到的遠少於 71。
- 檔案：`read_file`、`write_file`、`patch`（targeted edit）、`search_files`。Shell：`terminal`、`process_manage`。Browser：`browser_cdp/dialog/exec/vault`（需 Chromium，或 Portal 的 Browser Use 雲端）。另有 `execute_code`（Python 腳本以 RPC 呼叫 Hermes 工具）、`computer_use`、`web_search/web_extract`、`vision_analyze`、`todo_list`、`cronjob_manage`、`session_search`、`memory`、`skill_manage/skill_view/skills_list`、`image_generate`、`text_to_speech`、`clarify`。
- Composite toolsets：`coding`、`debugging`、`safe`；平台 toolsets 如 `hermes-cli`、`hermes-acp`、`hermes-telegram`。
- MCP：每個設定的 MCP server 變成 `mcp-<server>` toolset；v0.22.3 加入 MCP progress notification。另有 tool-search.md（動態工具搜尋，內容未讀）與 optional-mcps/ 目錄。
- Subagents（delegation.md verified）：`delegate_task`，預設最多 10 個並行；預設扁平（最大深度 1），可設 orchestrator 角色並提高 `max_spawn_depth`；子 agent 全新對話、自有 terminal session、只回傳 final summary；可選 `worktree_isolation`（git worktree 各自分支）；可設定較便宜的 `delegation.model`；子 agent 不能用 `memory`、`clarify`、`cronjob`、`send_message`。v0.22.5 提及 pluggable delegation backends（`delegation.backend`）。
- 驅動其他 coding agent：repo 有 `skills/autonomous-ai-agents/{claude-code,codex,opencode,computer-use,hermes-agent}` 目錄（目錄名 verified，內容未讀，具體做法 unverified）；另有 codex-app-server-runtime.md（未讀）。反向：Hermes 可作 ACP agent 被編輯器驅動（見 §10）。另可 `hermes import-agent` 匯入 OpenClaw、Claude Code、Codex CLI、Cursor 的設定/記憶/skills（README verified）。
- Kanban 多 agent 協作（14 個 kanban 工具、kanban-worker-lanes.md）、mixture-of-agents.md、LSP（lsp.md）存在，皆未深讀。

## 4. 學習迴圈（核心差異化）
來源：memory.md、skills.md、curator.md、sessions.md（均為 repo 內文件，verified；實際效果是否如文件所述 unverified）。

**Memory（宣告式記憶）**
- 兩個小檔：`MEMORY.md`（agent 筆記，約 2,200 字元上限）與 `USER.md`（使用者畫像，約 1,375 字元），位於 `~/.hermes/memories/`。超額時 tool 回錯，逼 agent 自行整併/刪除再重試，不會靜默丟棄。
- `memory` tool 動作：add / patch / remove（以唯一子字串比對，模糊比對會被拒絕，完全重複略過）；無 read 動作，內容於 session 開始時注入。
- **Frozen snapshot**：記憶只在 session 啟動時載入 system prompt，session 內寫入立即落檔但下個 session 才進 prompt（保 prompt cache）。這意味著「學到的東西」不會立即生效，文件建議在任務邊界用 `/new`。
- 觸發：每回合結束後 background review 可能存記憶或更新 skills；頻率由 `memory.nudge_interval`、`skills.creation_nudge_interval` 控制；可用較便宜模型、設 token 預算、關閉；本地 GPU 時預設等閒置才跑。
- 使用者控制：直接編輯檔案；`write_approval: true` 後寫入先暫存，用 `/memory pending|approve|reject` 審核；`hermes journey list|delete|edit` 修剪（文件稱刪除的 skill 進 archive、memory entry 直接移除）；可用 `memory_enabled`/`user_profile_enabled` 關閉。外部 provider 外掛（Honcho 辯證式 user modeling、RetainDB、ByteRover 等）可並存。
- 安全掃描：記憶項目檢查 prompt injection、憑證外洩、SSH 後門樣式、不可見 Unicode，命中則封鎖。

**過往對話搜尋**
- `session_search` 工具用 SQLite FTS5 全文檢索 `state.db`，sessions.md 稱「不呼叫 LLM、回傳實際訊息」，可捲動前後文或瀏覽近期 session（README 則寫 "with LLM summarization"，兩者矛盾，以較細的 memory.md/sessions.md 為準但標註 unverified）。repo 有 `native/fts5_cjk`，顯示有中日韓分詞處理（目錄名 verified，細節未讀）。

**Skills（程序性記憶）**
- 存放 `~/.hermes/skills/<name>/SKILL.md` + references/templates/scripts/assets；相容 agentskills.io 標準；採 progressive disclosure（先列清單，需要才載入全文）。
- 自動建立：agent 以 `skill_manage`（create / patch（偏好）/ rewrite / delete / write_file / remove_file）把摸索出的多步驟流程存成 skill；background review 於 session 後建議或更新。advisory linter 只警告（如本文過大）不阻擋寫入。
- 審核：`skills.write_approval: true` 時寫入暫存在 `~/.hermes/pending/skills/`，用 `/skills pending|diff|approve|reject`。預設是否需審核：文件只說可開啟，預設值本次 unverified（推論為預設不需審核）。
- **Curator**（curator.md）：背景維護，僅在閒置時觸發（預設間隔 7 天、閒置 2 小時）。階段一（無 LLM）：14 天未用標 stale、30 天未用移入 `.archive/`；階段二（opt-in）LLM 整併/patch/合併重疊 skills。只管 agent 自建的 skills；hub 安裝者豁免；有 cron 引用的 skill 受保護；可 pin。**從不自動刪除**，只 archive；有 `restore`、整併前全樹 snapshot（`hermes curator rollback`）、逐筆 audit ledger（`rollback <entry-id>`）、`purge` 需手動。使用統計在 `~/.hermes/skills/.usage.json`。
- Hub 安裝防護：來源分 builtin/official（信任）、trusted（如 openai/skills、anthropics/skills）、community；每次安裝跑內建安全掃描，dangerous 判定無法 `--force` 繞過；可 `inspect` 預覽、`audit`；本地改過的 bundled skill 不會被更新覆蓋。
- 使用者可檢視：直接開檔、`hermes skills list/inspect`、`/skills diff`、curator ledger。
- 風險面（推論）：自動建立的 skill 會變成之後每次對話可被載入的指令；掃描只涵蓋已知樣式；「錯的但無害」的 skill（過時流程、錯誤假設）沒有自動驗證機制，只靠使用頻率淘汰（不用才 archive，常用的錯 skill 反而留下）。文件未提到對 skill 品質的自動測試（unverified）。

## 5. Context 管理
- System prompt：由 `agent/` 的 prompt builder 組成（SOUL.md 身份 + memory snapshot + skills 清單 + 專案 context file 等；確切組裝順序 unverified）。SOUL.md 為全域人格檔，只從 `HERMES_HOME` 載入。
- Compression（developer-guide/context-compression-and-caching.md）：預設 `compression.threshold` 0.50（context window 的 50%，低於 512K 視窗時下限 0.75），四階段：先無 LLM 地把舊 tool 結果（>200 字元）換成佔位符；保護前 3 則訊息與尾端（lean 模式約視窗 2.5%、10K–25K tokens，至少 20 則訊息/1 則使用者訊息）；中段交給 `auxiliary.compression` 模型用結構化模板（Goal/Progress/Key Decisions/Relevant Files/Next Steps）摘要，後續壓縮是更新前一份摘要；失敗時有 deterministic fallback。注意：摘要模型 context 必須不小於主模型，否則中段被直接丟棄。Gateway 另有 85% 的 session hygiene。v0.22.3 新增 `post_compaction` plugin hook。
- 規則檔（context-files.md verified）：專案檔「第一個命中即用」：`.hermes.md`/`HERMES.md` > `AGENTS.override.md` > `AGENTS.md` > `CLAUDE.md` > `.cursorrules`（另讀 `.cursor/rules/*.mdc`）。**相容 AGENTS.md 與 CLAUDE.md**。git repo 內會合併從 git root 到 cwd 的 AGENTS.md 鏈（深層優先）；啟動只載 cwd 的，之後碰到其他目錄時才把該目錄檔案附在 tool result（最多往上 5 層，每檔 32K 字元）。單檔上限隨模型視窗（20K–500K 字元），超長 70% head/20% tail 截斷。
- 其他：`/goal`（Ralph 式迴圈，judge 模型判斷是否達標，預設最多 20 個續跑回合）、context-references.md（@ 引用，未讀）。

## 6. 權限與安全
（security.md verified）
- 核准模式 `approvals.mode`：**smart（預設）**——輔助 LLM 評估風險，低風險自動核准、危險自動拒絕、不確定才問使用者；manual；off。核准逾時視為拒絕；可選 once / session / always（加入永久 allowlist）。`--yolo`、`/yolo`、`HERMES_YOLO_MODE=1` 可跳過提示，但仍不繞過 hardline blocklist（如 `rm -rf /`、fork bomb、raw disk write）、MCP spend limits、MCP elicitation 與付款卡填寫提示。`approvals.deny` 可用 glob 定義永遠封鎖的命令（文件自承是命令政策而非 sandbox）。
- 檔案寫入保護：`write_file`/`patch` 不能碰 `~/.ssh/`、`.env`、Hermes 祕密庫；`HERMES_WRITE_SAFE_ROOT` 可限制寫入根目錄；文件自承 terminal 仍可繞過。
- Terminal backends：local、Docker、SSH、Modal、Daytona、Vercel Sandbox（README），外掛可加 Singularity/Apptainer。容器 backend 視容器為安全邊界，**略過 dangerous-command 檢查**；Docker 丟棄幾乎所有 capabilities、禁提權、限資源。預設 backend 是否為 local：推論為是（README 以 local 為首），本次未直接驗證。
- Gateway 授權：依序檢查 per-platform allow-all → DM pairing 核准 → 平台 allowlist → 全域 allowlist → 全域 allow-all；**未設任何 allowlist 時預設拒絕所有人**；DM pairing 為 8 字元碼、1 小時過期、有速率限制與鎖定。ACP 文件警告 Buzz 若設「Anyone」會讓所有頻道成員取得同等 shell 權限。
- Prompt injection / 供應鏈：專案 context file 掃描（指令覆寫、隱藏 HTML、憑證外洩、不可見字元），命中則專案檔不載入，使用者自己的 SOUL.md 命中只警告；文件註明不能取代人工審閱外部專案的 AGENTS.md。MCP 子行程只拿到安全環境變數白名單；URL 工具有 SSRF 防護；已安裝 Python 套件會對照已知被入侵版本清單；cron prompt 建立時掃描；cron session 預設不得再建 cron。plugin 的 `pre_tool_call` hook 逾時/錯誤時 fail closed。tirith 掃描器已移除（security.md 明載）。
- **已知缺口（issue）**：#141346（open，36 則留言，type/security）——smart approval 自動核准 `osascript` 指令，關閉使用者未存檔的 PowerPoint 並刪除唯一 PDF，資料無法復原；社群 PR 分支嘗試把 AppleScript 類指令導向人工審核，reviewer 指出可被 base64/awk 繞過，尚未合併。這顯示「用 LLM 當風險判官」的預設對新手並不安全（verified 為 issue 內容；影響程度為推論）。

## 7. 復原機制
（checkpoints-and-rollback.md verified）
- 有檔案層級 checkpoint：以 **shadow git repository**（`~/.hermes/checkpoints/store/`，每專案一個 ref `refs/hermes/<hash>`），不動使用者專案的 `.git`。在 `write_file`/`patch` 前與破壞性 terminal 命令（rm、mv、cp、sed -i、輸出重導向、git reset/clean/checkout）前建立，每目錄每回合最多一次，無變更則跳過。
- 指令：`/rollback`（列表含變更統計）、`/rollback <N>`（還原並保留使用者手改、同時 undo 上一個對話回合）、`--all` 完整覆蓋、`/rollback diff <N>`、`/rollback <N> <file>` 單檔；還原前先存 pre-rollback snapshot，可再回復；以 agent-write ledger（記錄 Hermes 最後寫入的檔案 hash）辨識手改。
- **限制**：v2 起為 **opt-in**（`checkpoints.enabled` 或 `--checkpoints`），預設不開；容器 backend（docker/modal/daytona/vercel_sandbox）無 checkpoint；巢狀 git repo 只存 gitlink；預設每專案 20 份、總量 500MB、>10MB 檔案與 >50,000 檔案目錄跳過；無 git 則停用；auto-prune 7 天。
- `/undo`：README 列出通用 slash command `/undo`，但 checkpoint 文件未提及，具體行為 unverified。
- 對話層級：session 可 resume、compression 產生編號續接；auto-prune 預設關。Skills 有 curator rollback，memory 項目刪除則無還原（文件稱「removed」）。

## 8. 模型與成本
- Provider：README 稱支援 Nous Portal、OpenRouter、OpenAI、自訂 endpoint 等，`hermes model` 切換；有 fallback-providers.md、provider-routing.md、credential-pools.md、subscription-proxy.md（皆未深讀）；`providers/` 目錄存在。完整 provider 清單 unverified。
- 本地模型（local-models.md verified）：llama.cpp 為主路徑，Hermes 可自動安裝/管理對應硬體的 build，或偵測既有 `llama-server`（預設只探測 8080）；Ollama 另有專頁；任何 OpenAI-compatible endpoint 可用；推薦模型至少 64K context，從不提供低於 4-bit 的量化；閒置 15 分鐘卸載。vLLM、LM Studio：頁面未提（unverified）。
- 成本：Desktop 狀態列顯示 context 使用量、cache hit rate、tokens/sec，切換到被標為昂貴的模型前會警告（desktop.md）；沒有專屬金額顯示的文件說明。`/usage`、`/insights` 為 slash command（README），內容未讀。預算控制：未找到全域花費上限（unverified）；有 goals 的 turn budget（預設 20）、delegation 並行/深度上限、review 的 token 預算、MCP spend limits。輔助任務（compression、goal_judge、背景 review）可各自指定較便宜模型。
- Nous Portal：訂閱制，含模型 + Tool Gateway，`hermes setup --portal`、`hermes portal info`。價格本次未查（unverified）。v0.22.5 提到「reworked Nous sign-in flow、usage-window plan support」。

## 9. 上手體驗
- 安裝（README verified）：Linux/macOS/WSL2 為 curl 腳本；原生 Windows 為 PowerShell 一行指令（宣稱完整支援）與 MSIX；Android/Termux 有 APT repo；Desktop App 支援 macOS（Apple Silicon）、Windows、Linux，已有 Hermes 者可 `hermes desktop`。基礎指令：`hermes`、`hermes setup`、`hermes model`、`hermes tools`、`hermes doctor`、`hermes update`。常見摩擦：防毒誤判 `uv.exe`（README 有專段）。
- Desktop onboarding：可「Choose provider later」；Basic/Elite 版面決定 Simple/Advanced 介面模式；內建檔案瀏覽、preview、整合終端、Git review/worktrees、記憶圖、排程、語音、Artifacts gallery。是否 beta：文件未述。
- 定位：通用個人 agent（跨 Telegram/Discord/Slack/WhatsApp/Signal/Email 等）為主，coding 是其中一個 toolset（`coding` composite、ACP、LSP、Git worktrees）。比 Claude Code/Codex 類專用 harness 更偏「長駐助手」；coding 專屬品質（diff review、edit 成功率）本次無數據（unverified）。
- 新手友善度評估（推論）：一行安裝 + Desktop + Portal OAuth 對新手門檻低；但表面積極大（71 工具、數十個設定頁、plugins、profiles、kanban、pets、skins…），Basic/Simple 模式是否有效收斂未驗證。

## 10. 擴充與自動化
- 排程（cron.md verified）：`/cron`、`hermes cron create` 或自然語言；格式含 `in 30m`、`every 2h`、`weekdays at 9am`、cron 運算式；可掛多個 skills；每次執行是全新 session；輸出可送回原聊天/檔案/多平台；`[SILENT]` 不送出、`[CRON_FAILURE]` 標失敗；另有 no-agent 腳本模式（不呼叫 LLM）。agent 型 job 需 gateway 運行，每 60 秒檢查。另有 heartbeat.md、loops.md（`/loop` 即時倒數，v0.22.3）。
- Hooks（hooks.md verified）：四類——gateway hooks（`~/.hermes/hooks/` 的 HOOK.yaml+handler.py）、plugin hooks（`pre_tool_call`、`post_tool_call`、`pre_llm_call`、approval、subagent 等事件）、shell hooks（config.yaml `hooks:` 區塊，可擋 tool call/注入 context）、outbound webhooks（簽章事件推送）。
- Headless/SDK：`batch_runner.py` 與 batch trajectory generation（研究用）；api-server.md 與 `hermes-api-server` toolset（內容未讀）；`hermes serve`。無獨立「SDK」明確文件（unverified）。
- ACP（acp.md verified）：`hermes acp`，支援 VS Code（ACP Client 擴充）、Zed、JetBrains、Buzz；呈現 chat、tool 活動、file diff、approval 提示；session 可 list/resume/fork；限制：不含 messaging 與 cronjob 工具，部分 host 會自動回答核准，預設工具集含 terminal/execute_code。
- Plugins / skills hub：plugins/（memory、context_engine、model-providers、kanban、image_gen 等 kind）、plugin-catalog/、optional-skills/、optional-mcps/；Skills Hub 來源含 official、skills-sh、well-known、url、GitHub taps、clawhub、lobehub、browse-sh。plugin 依賴有 consent prompt（v0.22.4）。

## 11. 優點 / 缺點 / 痛點
優點
- 學習迴圈有完整治理面：容量上限、frozen snapshot、write_approval 暫存審核、掃描、curator 只 archive 不刪除、ledger 可逐筆 rollback。
- 檔案 checkpoint（shadow git）與 session 搜尋、跨平台 handoff；多介面共用同一核心與狀態。
- AGENTS.md/CLAUDE.md/.cursorrules 相容、匯入其他 agent 設定；MIT；backends 與平台覆蓋廣；gateway 預設全拒。
- 發版快、社群巨大。

缺點
- 表面積過大、0.x 且高頻重構（v0.22.1 build 錯 commit 被 v0.22.2 取代；已知 0.22.1 建的 session 在 0.22.2 無法刪除；open issue #78647 godfile 重構 82 則留言）。
- Checkpoint 預設關閉、容器 backend 不支援；容器 backend 略過危險命令檢查。
- 預設 smart approval 依賴 LLM 判斷（#141346）。
- 學習結果預設不需審核（推論，見 §4）；memory 刪除無還原。
- 官方宣稱（README「only agent with a built-in learning loop」）為行銷語言，未見效果評測（unverified）。

新手痛點（issue 佐證）
- #141346：自動核准造成未存檔工作遺失。
- #107402：`hermes update` 後持續出現「did not restart running gateways」警告（P2，27 則留言）。
- #134107：打包的 provider 在 stripped PM runtime 載入失敗（39 則）。
- #127665：Desktop 同一回覆渲染兩次（58 則，P2）；#127647：Desktop 閒置資源耗用追蹤（31 則）。
- 防毒誤判 uv.exe（README）。

專業使用者痛點
- #18715：要求遠端 Hermes 搭配本地 tool 執行（31 則，needs-decision）。
- #97681：跨 gateway 的 Bot 協作（40 則）；#64231：hook taxonomy 與待處理 hook PR（30 則）；#131859：無法經 API 開 PR（51 則，upstream）。
- open issues/PR 皆「5k+」，維護負荷高；CI/PR 由 triage bot 標籤，審查資源可能吃緊（推論）。
（留言數、標籤來自 issues 列表頁摘要，反應數未取得，樣本為留言數前 12 名。）

## 12. 對我們的啟發
值得抄
1. **記憶分兩層**：小而有上限的常駐記憶（MEMORY/USER）+ 無上限但按需檢索的 session 全文搜尋（FTS5、不呼叫 LLM、免費）。強制容量上限讓 agent 必須整併，是低成本防膨脹法。
2. **暫存審核佇列**（pending → diff → approve/reject）：對新手應「預設開啟」並以 GUI 呈現「我學到了這個，要保留嗎？」。Hermes 預設關閉，這是可超越之處。
3. **Curator 式「只 archive 不刪除」+ ledger + 逐筆 rollback + usage 統計**；pin 機制。
4. **Frozen snapshot 保 prompt cache**，同時對使用者說明「下個 session 才生效」或提供 `--now`。
5. **Shadow git checkpoint**（不碰使用者 .git）+ 手改偵測 ledger + 還原前自動備份——但應預設開啟，並涵蓋容器/沙箱。
6. **規則檔相容**：AGENTS.md/CLAUDE.md/.cursorrules 讀取 + 注入掃描；子目錄 context 懶載入。
7. 匯入其他 agent 的設定/記憶（import-agent --dry-run），降低轉換成本。
8. 輔助任務（壓縮、judge、背景 review）可指定便宜模型；`/goal` 迴圈附 turn budget。
9. Gateway 預設全拒 + DM pairing；cron 禁止遞迴建 cron。

避免
- 用 LLM 風險判官當預設核准機制（#141346）；應以確定性規則 + 「GUI app 狀態、未存檔工作、刪除非自建檔案」升級人工確認；fail closed。
- 自動學習的狀態未驗證即被信任：建議對新建 skill/記憶標明來源（哪個對話、何時）、先「試用/草稿」狀態、附一鍵停用與回滾，並對關鍵指令類 skill 要求確認；定期顯示「本週學到什麼」給新手看。
- 學習狀態只在「不使用」時淘汰：錯但常用的 skill 不會被清除，需加入失敗回報/使用者否決訊號。
- 表面積爆炸（71 工具、數十設定頁）與 0.x 高頻破壞性重構；新手版應提供精簡預設工具集。
- 摘要模型小於主模型時中段被丟棄這類靜默失敗，應顯性警告。
- 記憶刪除不可還原；對新手應有垃圾桶。

## 13. 來源與未能驗證
來源（皆於 2026-10-26 取得，WebFetch 摘要）
- https://github.com/NousResearch/hermes-agent （首頁、releases、issues 列表、#141346、website/docs/user-guide 與 features 目錄、skills/autonomous-ai-agents 目錄）
- raw.githubusercontent.com/NousResearch/hermes-agent/main/ 下：README.md、AGENTS.md、website/docs/user-guide/{security,sessions,desktop,local-models,checkpoints-and-rollback}.md、website/docs/user-guide/features/{memory,skills,curator,cron,hooks,acp,delegation,goals,context-files}.md、website/docs/reference/toolsets-reference.md、website/docs/developer-guide/context-compression-and-caching.md
- 官方網站 hermes-agent.nousresearch.com 本次未直接抓取。

未能驗證
- 文件站渲染版；GitHub 語言比例；repo 首頁 Releases 區塊為空的原因；open issues/PR 確切數字。
- Nous 融資、Portal 價格與方案；provider 完整清單；vLLM/LM Studio 支援；全域花費預算控制；`/usage`、`/insights` 實際內容。
- `/undo` 行為；memory 刪除能否還原；skills 的 write_approval 預設值；Basic/Simple 介面模式實際收斂程度。
- session_search 是否使用 LLM 摘要（README 與 sessions.md 矛盾）。
- skills/autonomous-ai-agents/{claude-code,codex,opencode} 的實際驅動方式；codex-app-server-runtime、kanban、tool-search、lsp、api-server、heartbeat、loops、honcho 等頁面未讀。
- 原始碼層級：agent turn 階段、smart approval 風險判定實作、learning loop 實際成效（無第三方評測，不引用 blog roundup）。
- Issue 樣本僅留言數前 12 名，未取得 reaction 數與 Discussions/社群討論。
