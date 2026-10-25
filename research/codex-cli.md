# OpenAI Codex CLI 研究報告

檢查日期：2026-10-25。資料來源以 github.com/openai/codex 的 main 分支原始碼（raw.githubusercontent.com）、repo 頁面與 Releases 頁面為主。**developers.openai.com 在本環境被 proxy 封鎖（403/ENOTFOUND），官方文件頁無法直接讀取**；文件內容僅能透過 repo 內 docs/*.md（多為連結 stub）與 WebSearch 摘要間接取得，已於各處標註。

## 1. 基本資料

- Repo：github.com/openai/codex；授權 Apache-2.0（repo 頁面明載）。
- 技術棧：核心為 Rust（`codex-rs/` Cargo workspace，數十個 crate）；npm 套件 `@openai/codex` 為發行包裝；另有 TypeScript SDK（`sdk/typescript`，`@openai/codex-sdk`）。GitHub Languages 比例本次抓取為空，**未驗證**。
- 維護者：OpenAI（官方）。
- 與其他產品關係（README 明載）：CLI 本地執行；IDE 擴充（VS Code/Cursor/Windsurf）、桌面 app（`codex app`）、Codex Web（雲端 agent，chatgpt.com/codex）為不同入口。程式碼層面可見 `codex-rs/app-server`（供 IDE/app 使用的協議伺服器）、`cloud-tasks`、`cloud-client` 等 crate，顯示 CLI 同 repo 內也包含雲端任務客戶端。
- 活躍度：約 130.9k stars、20.6k forks、661 watchers（repo 頁面，2026-10 抓取）。Releases 頁：最新穩定版 0.164.0（Oct 23），同日有 0.165.0-alpha.1；Oct 20–23 四天內有近 10 個 alpha 預發行，即**幾乎每日 alpha、約每週穩定版**（依 10 筆列表推論，穩定版精確節奏未驗證）。
- 0.164.0 重點（Release notes）：`/presets`、`/model` `/permissions` 內嵌參數、終端圖片預覽、Bedrock 設定精靈預設啟用、compaction 失敗不再留下殘缺摘要。

## 2. 架構

- Workspace 規模大（Cargo.toml 列出 >100 個成員）。關鍵 crate：`core`（業務邏輯，README：「designed to be used by the various Codex UIs written in Rust」）、`tui`、`exec`（headless）、`cli`（multitool，含 `codex exec/review/login/mcp/app-server/sandbox/execpolicy/resume/fork/apply/cloud/doctor/features` 等子命令）、`app-server` + `app-server-protocol`（JSON-RPC）、`protocol`（Op/Event 型別）、`apply-patch`、`execpolicy`、`linux-sandbox`、`windows-sandbox-rs`、`sandboxing`、`network-proxy`、`rmcp-client`/`codex-mcp`、`hooks`、`exec-server`（遠端/多環境執行）、`model-provider-info`、`ollama`/`lmstudio`、`rollout`（session 落盤）、`memories`、`worktree`、`git-utils`。
- Agent loop（`codex-rs/docs/protocol_v1.md`）：UI 與核心以 **SQ/EQ（Submission Queue / Event Queue）** 溝通；實體為 Session → Task → Turn。Turn = 一次 Model 請求（Responses API，SSE 串流）＋執行命令/patch＋必要時暫停等待核准；Turn 無輸出則 Task 結束。一個 Session 同時最多一個 Task；平行工作建議多個 Codex 實例。`response_id` 存入 Session，可用於 resume/fork。文件註明「程式碼未必完全符合 spec」。
- App-server：README 顯示大量 JSON-RPC 方法（`thread/list`、`thread/read`、`thread/revert`、`windowsSandbox/setupStart`、MCP OAuth login、帳號/企業登入等），可 `codex app-server generate-ts` / `generate-json-schema` 產生型別。`approval-presets` 原始碼註解寫明「UI-agnostic，TUI 與 MCP server 共用」。
- 共用方式：TUI、exec、IDE/app 都是 `core` 的不同前端；非 Rust 前端透過 app-server 協議。TS SDK 則是 spawn CLI 並交換 JSONL（SDK README）。

## 3. 工具集

依 `core/src/tools/handlers/` 檔名（驗證存在，行為細節未逐一讀碼）：
- Shell：`unified_exec`（PTY、背景終端；TUI 有 `/ps`、`/stop`）、`shell_spec`。
- `apply_patch`：獨立 crate `apply-patch`，並以 `arg1 = --codex-run-as-apply-patch` 讓同一 binary 模擬虛擬 `apply_patch` CLI（core README）。具體 patch 格式（`*** Begin Patch` 一類）本次**未讀取確認**。
- Plan：`plan.rs`；TUI `/plan` 切換 Plan mode，protocol 有 `PlanDelta`。
- 其他：`view_image`、`request_user_input`、`request_permissions`、`tool_search`、`mcp_resource`、`multi_agents`/`multi_agents_v2`、`get_context_remaining`、`new_context_window`、`dynamic`（動態工具）、`request_plugin_install`、`sleep`、`current_time`。
- Web search：config_types.rs 有 `WebSearchMode` / `WebSearchContextSize` / `WebSearchLocation`；ext 目錄有 `web-search`、`image-generation`。預設值未驗證。
- MCP client：`rmcp-client` crate；`codex mcp list/get/add/remove/login/logout`。
- MCP server 模式：舊版有 `codex mcp-server`；本次檢視 main 的 CLI Subcommand 列表與 workspace 成員**未見 mcp-server**，疑似已移除或改名，**未驗證**，請以官方文件為準。
- 自訂工具：MCP、動態工具、plugins（`codex plugin`、marketplace）、skills、extensions（`ext/*`）。

## 4. Context 管理

- AGENTS.md（`core/src/agents_md.rs`，已讀）：自 project root（預設以 `.git` 為 marker，可由 `project_root_markers` 設定）向下到 cwd，逐層收集並依序串接；優先檔名 `AGENTS.override.md`，再 `AGENTS.md`，再 `project_doc_fallback_filenames`；使用者層與專案層以 `--- project-doc ---` 分隔；受 `project_doc_max_bytes` 限制；不越過 project root。`/init` 可產生 AGENTS.md。
- Compaction：`core/src/compact.rs` 含 inline auto compact 與手動 `/compact`；有 remote compaction（`compact_remote_v2*`）；摘要以 `SUMMARY_PREFIX` 前綴插入歷史，並保留使用者訊息。另有 `new_context_window`、`get_context_remaining` 工具。
- Sessions：rollout 落盤（TS SDK README 稱存於 `~/.codex/sessions`）；`codex resume`、`codex fork`、`archive`/`delete`；TUI `/resume` `/fork` `/rename` `/export`。另有 memories（`/memories`，標示實驗性）、`/recap`、`/goal`（長任務目標）。
- Sub-agents：確有實作證據：`multi_agents` handlers、`agent-roles` crate、TUI `/agents`、`/multi-agents`、hook 事件 `SubagentStart/Stop`、`SubagentHookContext`。使用體驗與預設是否開啟**未驗證**。

## 5. 權限與安全

- 兩軸設計：approval policy × sandbox。
  - `AskForApproval`（protocol.rs 已讀）：`untrusted`（UnlessTrusted：除非 execpolicy 規則允許才問）、`on-request`（預設，由模型決定何時問；別名 `on-failure`）、`granular`（可逐類別開關：sandbox_approval、rules、skill_approval、request_permissions、mcp_elicitations；為 false 時自動拒絕而非詢問）、`never`。
  - `SandboxPolicy`：`read-only`、`workspace-write`（可設 `writable_roots`、`network_access`、是否排除 TMPDIR）、`danger-full-access`、`external-sandbox`（已在外部沙盒中）。網路預設關閉。
  - 內建 preset（approval-presets 原始碼）：Read Only（on-request + 唯讀）、Default/Agent（on-request + workspace-write）、Full Access（never + 無沙盒）。旗標 `--dangerously-bypass-approvals-and-sandbox`（`--yolo`）與 `--full-auto`（workspace-write + on-request）見 WebSearch 摘錄官方 security 頁（二手，未直讀）。
  - 新的 `[permissions]` permission profile / 分離式 filesystem policy（可 `none` 拒讀、glob 遮罩如 `"**/*.env" = "none"`）；另有 `auto_review`（Guardian 自動審核，`TooManyDenials` 斷路器）。
- execpolicy：Starlark `prefix_rule(pattern, decision=allow|prompt|forbidden, justification, match, not_match)` 與 `host_executable`；`codex execpolicy check`；規則示例會在載入時驗證（類似單元測試）。
- 各 OS 實作：
  - macOS：`/usr/bin/sandbox-exec`（Seatbelt）；workspace-write 時 `.git`（含 gitdir 指標目標）與 `.codex` 保持唯讀。
  - Linux：預設 bubblewrap（`--ro-bind / /`，可寫根目錄再 `--bind`，`--unshare-user`）+ `PR_SET_NO_NEW_PRIVS` + seccomp（受限網路時封鎖網路 syscall、封鎖 `AF_VSOCK`、一律封 `io_uring`）；舊 Landlock 路徑保留為 legacy，且對需要精細 filesystem 限制的 policy 會被拒絕；找不到 `bwrap` 則用隨附版並顯示啟動警告；無法建立 user namespace 亦警告；**WSL1 不支援**（WSL2 正常）。符號連結與不存在的受保護路徑以 `/dev/null` 掛載封鎖。
  - Windows：兩種 legacy 後端（elevated / unelevated restricted-token）與較新的 `mxc`（`mxc-sandbox`）；無法被後端強制執行的 policy 會 fail closed。`/setup-elevated-sandbox`（ElevateSandbox）需要提權設定。
- 網路政策：`network-proxy` crate 提供本地 HTTP(3128)/SOCKS5(8081) proxy，支援 allow/deny 與 `limited`（唯讀）模式，limited 時 HTTPS MITM，本地/私有網路預設拒絕。
- 已知缺口（來自 issue，WebSearch 摘要，未逐一開啟）：Windows 上 #51799（反覆核准、elevated sandbox 佈建失敗）、#52735（佈建因 ACL 自身執行檔失敗，唯有 Full access 才能用）、#55057/#53134/#54368（選「Ask for approval」卻因 `Granular.sandbox_approval=false` 拒絕 write_stdin）、#19315（git worktree 因 `.git` 唯讀需反覆核准）、#52845（Windows 回歸群集）。
- 評價（推論）：沙盒優先設計對專業者與 CI 很強；對新手，預設 workspace-write + 無網路可擋住多數災難，但「approval 與 sandbox 兩軸」概念複雜，Windows 設定失敗時新手只能退到 Full access，風險反而更高。

## 6. 復原機制

- 已驗證：TUI 有 `/diff`、`/review`、`codex apply`（套用雲端任務 diff）、`/fork`、app-server `thread/revert`（舊 `thread/rollback` 已移除）；Esc 回溯僅回退對話，**不還原檔案**（issue #11626 描述）。
- **未見官方檔案級 undo/checkpoint**：issue #9203「Please make /undo back」（其他多數被標為其重複）、#16784（要求恢復 Undo/Redo）、#19205（Undo 不應依賴 git）、#5082（VS Code Undo 會 stage 到 git）、#27636（open，rewind 工作流）、#55742。顯示過去曾有 undo 後被移除，目前主要依賴 git。TS SDK 與 exec 預設要求工作目錄為 git repo（`--skip-git-repo-check` 可略過），理由是「避免不可復原的錯誤」。
- 結論：復原＝git + 沙盒 + 唯讀 `.git` 保護；對不使用 git 的新手是明顯缺口。

## 7. 模型與成本

- Provider（`model-provider-info/src/lib.rs`）：內建 `openai`、`ollama`、`lmstudio`（OSS，`--oss`/`--local-provider`）、`amazon-bedrock`、`amazon-bedrock-runtime`、`gcp`（Vertex AI）；可自訂 provider，`wire_api = "chat"` 已移除，僅支援 Responses API（討論 #7782）。故「非 OpenAI 模型」支援有限：自訂 provider 必須講 Responses API。模型清單（含 gpt-5.x/6 系 Bedrock ID）以原始碼為準，未逐一驗證可用性。
- 認證：ChatGPT 登入（README 推薦，含 Plus/Pro/Business/Edu/Enterprise）與 API key；`codex login/logout`；ChatGPT 模式 base URL 為 `chatgpt.com/backend-api/codex`；另有企業 SSO、gateway OAuth、AWS 認證。
- 成本可視性：`/status`（設定與 token 使用）、`/usage`（帳戶用量/額度重置）、exec JSONL 的 `turn.completed` 含 usage；實際金額顯示**未驗證**。

## 8. 上手體驗

- 安裝：`curl -fsSL https://chatgpt.com/codex/install.sh | sh`、PowerShell 腳本、`npm i -g @openai/codex`、`brew install --cask codex`、GitHub Release 二進位（macOS arm64/x64、Linux x64/arm64）。
- 首次執行：`codex` → 登入選擇（ChatGPT 或 API key）。信任專案（Project trust）流程見 app-server README。`codex doctor` 診斷；`/init` 產生 AGENTS.md；`/import` 可匯入 Claude Code 設定與近期對話（值得注意的競品遷移功能）。TUI 設定豐富（`/keymap`、`/vim`、`/theme`、`/statusline`）。首次互動細節（畫面、引導）**未實測**。

## 9. 擴充與自動化

- `codex exec`（別名 `e`）：stdin/參數 prompt、`--json` JSONL、`--output-schema`、`-o` 輸出最後訊息、`--ephemeral`、`--ignore-user-config`、`--ignore-rules`、`--skip-git-repo-check`、`--strict-config`、`exec resume`；`codex review`。
- 設定：`~/.codex/config.toml`（docs/config.md 指向官方 config-basic/advanced/reference）；`-c` 覆寫；permission profiles（`default_permissions`）；requirements.toml 供管理員強制設定（如 `allow_managed_hooks_only`）；`--strict-config`。Profiles 的具體語法**未直讀官方文件**。
- Hooks：已驗證存在，14 個事件：PreModelRequest、PostModelRequest、PreToolUse、PermissionRequest、PostToolUse、PreCompact、PostCompact、SessionStart、SessionEnd、UserPromptSubmit、SubagentStart、SubagentStop、Stop、Interrupt；`/hooks` 管理；另有 legacy notify。
- Skills（`docs/skills.md`、`/skills`）、plugins/marketplace、自訂 prompts（`/presets` 10 個槽）。
- SDK：TypeScript（spawn CLI，JSONL；`startThread/run/runStreamed/resumeThread`、結構化輸出、圖片）。GitHub Action、Python SDK 是否存在：**未驗證**。
- 遠端：`codex exec-server`、`remote-control`、`app-server daemon`、`cloud`。

## 10. 優點 / 缺點 / 痛點

優點：真正的 OS 級沙盒（Seatbelt/bwrap+seccomp/Windows）；協議層分離乾淨；單一 Rust binary；execpolicy 可測試；hooks 完整；多前端共用 core；Apache-2.0。
缺點：概念多（approval×sandbox×profile×feature flags），repo 迭代極快且 API/設定頻繁變更（多處「removed」「legacy」）；對非 OpenAI 模型支援弱；`mcp-server` 模式現況不明。
新手痛點：Windows 沙盒佈建失敗與重複核准（#51799、#52735、#55057 等）；無檔案級 undo（#9203、#16784）；非 git 目錄受限（#19205）。
專業者痛點：approval 疲勞（#19315 worktree 因 `.git` 唯讀需反覆核准）；Windows 回歸（#52845）；Guardian 自動審核誤判/逾時（#55036）；需檔案/對話雙軌 rewind（#11626、#27636）。
（issue 內容皆來自搜尋摘要，未逐一開啟全文。）

## 11. 對我們的啟發

可複製：
1. 沙盒預設開啟 + 預設無網路 + `.git`/設定目錄唯讀保護；提供一鍵三檔 preset（唯讀/預設/完全），隱藏 approval×sandbox 兩軸給新手。
2. SQ/EQ + 型別化協議，核心與 UI 分離，自動產生 TS 型別，讓 TUI、IDE、SDK 共用。
3. execpolicy 規則附 `match/not_match` 範例自我測試；`forbidden` 規則附替代建議（對 agent 友善）。
4. Hooks 事件表完整；`/import` 吸收競品使用者。
5. `codex doctor` 與啟動警告（bwrap 缺失）。
需避免：
1. 沙盒在 Windows 失敗時只能退到 Full access——應提供降級但仍有保護的路徑（如 git 自動快照）。
2. 不要像 Codex 一樣缺乏檔案級 undo；新手向產品應內建自動 checkpoint（不依賴使用者 git、不污染 staging）。
3. 設定面膨脹與頻繁破壞性變更；granular approval 設定與 UI 顯示不一致造成「選了詢問卻直接拒絕」。
4. 綁定單一 wire API 會限縮 provider 生態。

## 12. 來源 與 未能驗證

來源（皆主要來源，抓取於 2026-10-25）：
- https://github.com/openai/codex （repo 頁、releases、docs 目錄）
- https://github.com/openai/codex/tree/main/codex-rs 與 core/src、core/src/tools/handlers、tools/src、cli/src 目錄列表
- raw.githubusercontent.com/openai/codex/main/：README.md、codex-rs/core/README.md、codex-rs/linux-sandbox/README.md、codex-rs/network-proxy/README.md、codex-rs/execpolicy/README.md、codex-rs/app-server/README.md、codex-rs/docs/protocol_v1.md、codex-rs/protocol/src/protocol.rs、codex-rs/core/src/agents_md.rs、compact.rs、codex-rs/hooks/src/lib.rs、model-provider-info/src/lib.rs、cli/src/main.rs、tui/src/slash_command.rs、utils/approval-presets、exec/src/cli.rs、sdk/typescript/README.md、docs/*.md
- 二手（僅用於 issue 與官方 security 頁摘要）：WebSearch 結果，含 github.com/openai/codex/issues/{9203,11626,12558,16784,19205,5082,27636,55742,51799,52735,55057,53134,54368,19315,52845,55036}、developers.openai.com/codex/security、/concepts/sandboxing（僅取搜尋摘要）。

未能驗證：
- developers.openai.com 全部官方文件（proxy 封鎖）：security、cli/features、config-reference、profiles、GitHub Action、Python SDK、成本/定價說明。
- `codex mcp-server` 是否仍存在；web search 預設值；apply_patch 具體格式；sub-agent 預設啟用狀態；GitHub 語言比例與精確穩定版發佈節奏（API 403）；首次執行實際畫面；issues 全文與維護者回應；上述 issue 編號之狀態僅依搜尋摘要。
