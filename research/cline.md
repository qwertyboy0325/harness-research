# Cline 研究報告

查核日期:2026-10-25。標記 [V] = 來自一手來源(GitHub repo / repo 內 docs / releases);[I] = 推論;unverified = 未能確認。

## 1. 基本資料
- [V] Canonical repo:github.com/cline/cline;描述為 "Autonomous coding agent as an SDK, IDE extension, or CLI assistant"。
- [V] License:Apache-2.0;版權持有 Cline Bot Inc.。
- [V] Stack:TypeScript monorepo,Bun 為唯一 package manager,Node 22+。VS Code extension 套件名 `claude-dev`(webview UI、gRPC/proto);Desktop app 為 Tauri v2 (Rust) + Next.js + Bun sidecar。語言比例 unverified(頁面未顯示)。
- [V] 人氣:約 70.8k stars、7.7k forks(2026-10-25)。
- [V] 發版節奏:極高頻。近期:v4.1.24 (Oct 13)、v4.1.25 (Oct 16)、CLI v3.0.73 / SDK v0.0.95 / Desktop v0.0.48 (Oct 17),即約每週多個版本,且 extension / CLI / SDK / Desktop 各自獨立版號。
- [V] 商業模式:軟體開源,收入來自 Cline Provider(pay-as-you-go 模型轉售,100+ models)、ClinePass(US$9.99/月 flat-rate,開放權重 coding 模型,有 5 小時/週/月用量上限)、限時免費模型(其用量可能用於改進模型)、Cline Cloud sessions(託管 sandbox,逐步推出)、enterprise-solutions 文件。融資金額 unverified。

## 2. 架構
- [V] 已從 v3 單體 extension 重寫為 SDK 分層:`@cline/shared`(型別、tool 建立 helper、hook engine)→ `@cline/llms`(provider gateway)→ `@cline/agents`(stateless agent loop:迭代、tool orchestration、hook、runtime events、每次呼叫前的 turn preparation)→ `@cline/core`(sessions、持久化、內建工具、config discovery、RPC)→ `@cline/sdk`(re-export)。
- [V] Hub daemon:一個 detached daemon 以 WebSocket 服務本機所有 client(CLI/Desktop/extension),discovery record 內含隨機 auth token(owner-only 權限),負責 sessions、events、approvals、schedules;client 可 attach/detach 而不中斷 session。
- [V] Session 儲存:`~/.cline/data/sessions/`,索引在 `~/.cline/data/db/sessions.db`;完整 transcript 以 messages artifact 儲存,compaction 狀態另存 `${sessionId}.compaction.json`(含前綴 hash 驗證);subagent/team session 與 root 同表。root session 在第一個 user turn 被接受後才建立。
- [V] 版本演進:CHANGELOG 顯示 4.0.0 遷移到 SDK,4.0.1 因回歸而回滾,4.1.0 將 legacy 與 SDK 版本打包並分階段 rollout。[I] 代表 v3 舊程式碼路徑仍可能並存,是遷移風險。

## 3. 工具集
- [V] 內建 9 個工具(ClineCore):`read_files`、`search_codebase`、`run_commands`、`fetch_web_content`、`apply_patch`、`editor`、`skills`、`ask_question`、`submit_and_exit`;另有 `spawn_agent`、`tasks`(todo/scheduled)。
- [V] 檔案編輯:`editor` 與 `apply_patch` 二選一;Act mode 下 GPT/Codex 與 openai-native provider 用 `apply_patch`,其餘用 `editor`;Plan mode 兩者皆無;SDK 可用 `toolRoutingRules` 覆寫。這是依模型訓練偏好切換 edit 格式的實例。
- [V] Shell:`run_commands`,即時監看輸出(VS Code 版使用 terminal 整合,細節 unverified)。
- [V] 瀏覽器:SDK 工具清單中無 browser tool,僅 `fetch_web_content`;README 稱 VS Code extension 可 "browse the web",其 computer-use 式 browser 自動化在新版是否保留 unverified。
- [V] 搜尋:僅 `search_codebase`(extension 內含 ripgrep binaries)。無 embedding index 之說,[I] 以 grep 式為主。
- [V] 自訂工具:SDK 以 `createTool` 定義;plugins(CLI `cline plugin` 從 URL/npm/git/本地安裝),plugin 只在 worker thread 或 subprocess sandbox 執行(SDK v0.0.94)。
- [V] MCP:stdio、Streamable HTTP(建議)、SSE(legacy,未指定 type 時的預設);設定檔 `~/.cline/data/settings/cline_mcp_settings.json`;`cline mcp` wizard;每 server 有 `autoApprove` 清單;大型 MCP 輸出會分頁/快取。MCP marketplace:mcp-overview 文件未提及(unverified);release notes 提到 marketplace plugins。

## 4. Context 管理
- [V] System prompt:CLI 可用 `-s/--system` 覆寫;內容細節 unverified(未讀 prompt 原始碼)。
- [V] 規則:`.clinerules/` 或 `.cline/rules/`(workspace)、`~/Documents/Cline/Rules` 與 `~/.cline/rules`(global);Markdown;自動偵測 `.cursorrules`、`.windsurfrules`、`AGENTS.md`(含 `~/.agents/AGENTS.md`);YAML frontmatter `paths` glob 做條件式規則;Rules 面板可逐條開關;workspace 優先於 global。另有 `.clineignore`、skills(也發現 `.claude/skills`)、workflows。
- [V] Plan/Act:Plan 唯讀、不可改檔或執行命令;Act 可執行;切換保留對話;可為兩模式指定不同模型;`/deep-planning` 用於大任務。
- [V] 壓縮:Auto Compact 在接近 context window 時由 LLM 摘要並取代歷史,以 tool call 形式顯示且計費,重用 prompt cache;不支援的模型退回規則式 truncation。CLI `--compaction <agentic|basic|off>`。4.1.19 起以 provider 回報 token 數觸發;本地模型輸出達上限時 compact 後重試一次。可用 checkpoint 或編輯先前訊息回到摘要之前。具體門檻 unverified。
- [V] Repo 理解:靠 `search_codebase` / `read_files` 按需探索,[I] 無預建索引。
- [V] Sub-agents:`spawn_agent`,獨立 context、可平行、可巢狀、不取得 MCP 工具;僅 CLI 與 Desktop,VS Code/JetBrains 不支援;核准 `spawn_agent` 即核准其所有後續動作(approval 不繼承/不適用於內部)。另有 agent teams (`--team-name`)。

## 5. 權限與安全
- [V] 預設每個檔案編輯與命令都需核准。Auto-approve 以類別細分:讀/寫專案檔、讀/寫 workspace 外檔、safe 命令/所有命令、browser、MCP、notifications;「all」選項需基礎開關同時開啟。文件建議只預設開「讀專案檔」。
- [V] 危險命令:沒有固定 allowlist/blocklist,由模型為每個命令標 `requires_approval`;文件明言範例不保證。自動核准的終端命令執行滿 30 秒會通知。[I] 安全性取決於模型判斷,是 prompt-injection 風險面。
- [V] YOLO mode:全部自動核准(含 Plan→Act 轉換),無確認對話框。CLI 預設 `--auto-approve true`(ACP 模式為 false)——也就是 CLI 的 Act 模式預設即自動核准。
- [V] 4.1.15/4.1.16 起 MCP 改為單一全域開關,移除 per-tool 勾選(粒度退化)。

## 6. 復原機制(Checkpoints)
- [V] 實作:獨立的 shadow Git repo,每次 tool use 後 commit 一次工作區完整狀態;不動使用者主 repo;涵蓋未被 Git 追蹤的檔案;跨編輯器 session 保留。
- [V] 還原選項:Restore Files、Restore Task Only(刪後續訊息)、Restore Files & Task;另有 Compare(diff)與編輯舊訊息時 "Restore All"。
- [V] 限制:大型 repo 可能耗儲存並變慢,可關閉;文件未說明數量上限、保留策略、清理方式。4.1.19 加入持久索引以略過已見過的 untracked 檔;4.1.17 修正還原會丟 commit 的問題,現在若 checkpoint 之後已有 commit 會拒絕還原。YOLO 無額外 per-action 保護。VS Code/JetBrains 有;CLI 是否有 checkpoint unverified。

## 7. 模型與成本
- [V] Providers:Anthropic、OpenAI、Google、OpenRouter、Bedrock、Vertex/Azure、Groq、Cerebras、Ollama、LM Studio、任何 OpenAI-compatible;release notes 另提 Codex 登入、ai&、CommandCode 等。本地模型支援(有專屬文件夾 running-models-locally 與對本地模型長請求逾時的修正)。
- [V] 成本:壓縮文件確認 API 成本顯示於每次呼叫;4.1.16 對訂閱計費的 provider 隱藏成本估算。預算上限控制(hard budget cap)unverified。
- [V] 預設模型頻繁更換(Sonnet 5.5 → Gemini 4 Argon → 又回 Sonnet 5.5,數日內)。[I] 預設值變動快,使用者成本不穩定。
- [V] Hosted 層:Cline Provider、ClinePass、免費輪替模型(見第 1 節)。

## 8. 上手體驗
- [V] 五條路:Desktop(macOS,Windows beta)、IDE extension(VS Code/Cursor/Windsurf/VSCodium/Antigravity/JetBrains)、CLI(`npm i -g cline` → `cline auth` → `cline "task"`,Node 20+)、Kanban 預覽(`npx kanban`)、SDK。
- [V] 初學者必須先選 provider 並取得 key 或登入 Cline 帳號(Cline Provider / ClinePass / BYOK)。首次流程:安裝 → 側邊欄開啟 → 設定 provider → 開始任務。
- [I] 入口過多(5 種介面、3 種計費)對新手是認知負擔。

## 9. 擴充與自動化
- [V] Hooks:生命週期 hooks(含 UserPromptSubmit、TaskStart),定義在 `@cline/shared` hook engine;docs/customization/hooks.mdx 僅指向 SDK Plugins 頁,事件完整清單 unverified;hook 失敗預設忽略(`hookErrorMode: "throw"` 可改)。
- [V] CLI headless:`cline "prompt"`、stdin pipe、`--json` JSON lines(`agent_event`/`hook_event`/`team_event`/`run_result`/`run_aborted`)、`--timeout`、`--retries`(預設 3)、`--worktree`、`--id` 續跑、`--acp` 供編輯器整合;退出碼文件未載(unverified)。
- [V] 排程 (`cline schedule`,cron)、Connectors(Telegram/Slack/Discord/Google Chat/WhatsApp/Linear)、agent teams、`cline history export`。
- [V] GitHub 整合:Cline Cloud sessions 可針對指定 repo/branch 在託管 sandbox 執行;現成 GitHub Action/PR bot unverified。

## 10. 優缺點與痛點
優點 [V/I]:Apache-2.0;完整 SDK 分層(可嵌入);多介面共用 hub;checkpoint 對新手友善;Plan/Act 概念清楚;相容 AGENTS.md/.cursorrules/.claude/skills;provider 中立。
缺點:
- 安全依賴模型自行標記 requires_approval;無硬性 blocklist。
- 發版與預設模型變動極快;4.0 SDK 重寫曾回滾(CHANGELOG 4.0.1),穩定性風險。
- Subagent 一次核准即放行全部內部動作。
- 經濟模式混合(免費模型資料可能用於訓練改進)。
GitHub issue 實例(僅標題,內文未讀):#1146「Executing terminal commands hangs cline」、#531「Execute command just freezes」、#1157「API Request loading indefinitely」、#2175「Diff Edit Mismatch」、#1418「deepseek R1 on local ollama cannot use tools」、#1959「MCP servers: no way to see logs」、#3510「Licensing Conflict: Apache 2.0 vs ToS」。
新手痛點 [I,部分有 issue 佐證]:要先理解 provider/key/model;弱模型工具呼叫失敗(#1418、#5915);YOLO/auto-approve 風險說明太淡;成本不透明。
專業痛點:terminal 整合卡死、diff 編輯 mismatch、MCP 除錯、大型 repo checkpoint 效能(4.1.19 修正過)、多介面功能不齊(subagents 僅 CLI/Desktop)。

## 11. 生態
- Roo Code:原名 Roo Cline,fork 自 Cline。[V] 官方 repo RooCodeInc/Roo-Code 顯示 "archived by the owner on May 15, 2026",24.3k stars。分裂原因僅見二手來源(自訂 modes、diff 編輯降成本、更多 provider、節奏較快),屬 unverified。「官方推薦遷移至 Cline」亦屬二手說法。
- Kilo Code:二手來源稱 fork 自 Roo Code 並合併上游,後續部分基於 OpenCode 重建(unverified,未讀 Kilo repo)。
- [I] 開源 fork 能複製功能但需自行承擔上游合併成本;Roo 封存顯示 fork 路線的生存風險。

## 12. 對我們的啟發
可複製:
1. Agent loop stateless + 外層 core 管狀態 + 本機 hub daemon,讓 CLI/IDE/Desktop 共用 session。
2. Shadow-git checkpoint(每次 tool use 一次、涵蓋 untracked、三種還原粒度);對 vibe coder 是最重要的安全網,預設開啟。
3. Plan/Act 分離,且可為兩者配不同模型。
4. 依模型切換 edit 工具(editor vs apply_patch),附覆寫規則。
5. 相容 AGENTS.md/.cursorrules/.claude/skills,降低遷移門檻;條件式 glob rules。
6. JSON-lines 輸出的 headless 模式,利於 CI 與測試。
7. Compaction 以 LLM 摘要並以可見 tool call 呈現,可回溯。
避免:
1. 安全判斷全交給模型;應加規則式(路徑/命令 pattern)硬防線與 sandbox。
2. CLI 預設自動核准;初學者介面應預設保守並用白話解釋風險。
3. 入口與計費模式太多、預設模型頻繁變動;初學者只應看到「一鍵開始」。
4. 大改寫上線後回滾(4.0.1);用 feature flag / 並行版本策略而非整包切換。
5. Subagent 授權不繼承卻整批放行;應有範圍與預算限制。
6. Checkpoint 無保留/清理策略;需設計容量上限。
7. 缺少 hard budget cap(未確認其存在),我們應內建。

## 13. 來源
已讀(WebFetch):
- https://github.com/cline/cline
- https://github.com/cline/cline/releases
- https://github.com/cline/cline/tree/main/{docs, sdk, docs/tools-reference, docs/features, docs/customization, docs/cli, docs/getting-started, docs/core-workflows, docs/mcp, docs/sdk}
- https://github.com/cline/cline/issues(依 reaction 排序,僅標題)
- https://github.com/RooCodeInc/Roo-Code
- https://raw.githubusercontent.com/cline/cline/main/{README.md, AGENTS.md, CHANGELOG.md(前 100k 字元), sdk/ARCHITECTURE.md}
- https://raw.githubusercontent.com/cline/cline/main/docs/{tools-reference/all-cline-tools, features/auto-approve, features/auto-compact, features/subagents, customization/hooks, customization/cline-rules, cli/cli-reference, core-workflows/checkpoints, core-workflows/plan-and-act, getting-started/clinepass, getting-started/installing-cline, getting-started/free-models, mcp/mcp-overview}.mdx
- WebSearch(僅作二手線索):Roo/Kilo fork 相關結果(fast.io、morphllm.com、blog.kilo.ai 等,未全文閱讀)

未能驗證:
- docs.cline.bot 無法連線(DNS 失敗),未讀官方文件站,只讀 repo 內 mdx。
- 主要語言比例;融資金額與公司細節。
- System prompt 實際內容;Auto Compact 觸發門檻。
- Browser 自動化工具在新版是否存在;VS Code terminal 整合實作。
- MCP marketplace 現況;hook 事件完整清單;CLI 退出碼;CLI 是否有 checkpoint。
- GitHub Action / PR bot 整合;hard budget cap。
- Roo/Kilo 分裂的原始動機(只有二手說法);Kilo 與 OpenCode 關係。
- Issue 內文與目前開放狀態(僅讀標題/狀態)。
