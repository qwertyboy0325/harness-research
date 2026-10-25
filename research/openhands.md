# OpenHands 研究報告

查證日期：2026-10-25。docs.openhands.dev / docs.all-hands.dev / arxiv.org 在本環境 DNS 無法解析，因此文件站與論文內文**未能直接讀取**；以下事實皆來自 GitHub 頁面與 raw 原始碼。標記「推論」者為本人判斷。

## 1. 基本資料
- **Canonical repo**：github.com/OpenHands/OpenHands（原 All-Hands-AI/OpenHands，已更名；verified：頁面顯示此組織）。約 91.8k stars、12.2k forks（頁面顯示）。
- **重要變化（verified）**：主 repo README 已改以 **Agent Canvas** 為產品（「self-hosted developer control center for coding agents and automations」），前端 React/TypeScript（vite、react-router、electron、playwright、helm chart）。Python 的 agent、tools、server 已拆到獨立 repo **OpenHands/software-agent-sdk**（約 1.3k stars，MIT）。舊的 Python 單體（V0）不再是主 repo 內容；benchmarks README 提到「migrating from V0 to V1 Agent SDK」。
- **CLI**：OpenHands/OpenHands-CLI（MIT，267 stars）README 明說**不再積極維護**，建議改用 Agent Canvas。
- **授權**：主 repo 與 SDK repo 的 LICENSE 頁面標示 MIT；主 repo 頂層列表**未見 enterprise/ 目錄**（舊版 repo 曾有 enterprise 目錄與非 MIT 授權，此次未在目前樹中見到，舊況未驗證）。
- **公司／商模**：All Hands AI（推論：由 OpenHands 組織名與舊 repo 名得知）。README 稱 OpenHands Cloud 為 commercial offering，另有 OpenHands Enterprise infrastructure（自架）。價格未驗證。
- **活躍度**：頁面可見 release-please、CHANGELOG、Docker 映像 tag 1.33.0，issue 到 #17251 以上，屬高度活躍。

## 2. 架構
- **分層（verified）**：SDK（`openhands-sdk`：LLM/Agent/Conversation/Tool）→ `openhands-tools` → `openhands-workspace` → `openhands-agent-server`（REST + WebSocket）→ OpenAPI contract → TypeScript client → Agent Canvas UI。另有獨立 automation repo 負責排程、webhook、run history。SDK README 稱它是 CLI 與 Cloud 的引擎。
- **事件流**：Conversation 維護 append-only 的 `state.events`，持久化於 conversation 的 `events/` 目錄（AGENTS.md）；condensation 也是以 `Condensation` marker event 追加，由 `View` 套用後產生 LLM 所見事件（condenser README）。文件未稱其為 event sourcing，但行為相當（推論）。
- **Agent loop**：`LocalConversation.arun()` 驅動 agent `astep()` 呼叫 LLM；`interrupt()` 可取消並發出 interrupt event。
- **Runtime/workspace**：`openhands-workspace` 含 agent_sandbox、apptainer、cloud、docker、remote_api；也可直接用本機。主 repo 的 `OH_CONVERSATION_RUNTIME=docker` 讓每個 conversation 一個容器。git-backed 本機 workspace 會建 per-conversation worktree（`/tmp/conversation-worktrees/`）。
- **CodeAct**：SDK 目錄中未見專屬 CodeAct 模組；V1 以 tool-calling（Terminal / FileEditor 等）為主。CodeAct 設計在 V1 的現況**未驗證**。

## 3. 工具集（verified，目錄列表）
`terminal`、`file_editor`、`planning_file_editor`、`apply_patch`、`glob`、`grep`、`browser_use`、`task_tracker`、`task`、`delegate`、`child_conversation`（子 agent）、`workflow`、`gemini`、`ask_oracle`、`tom_consult`、`preset`。**IPython 工具未在 V1 目錄中見到**（V0 有，V1 狀態未驗證）。MCP：`sdk/mcp`，`mcp_config` 為 server 名稱到 `MCPServer` 的 flat map，於 FastMCP 邊界才轉換。Skills：AgentSkills 採 progressive disclosure，`<available_skills>` 區塊，描述截斷 1024 字；`load_public_skills=True` 可載入公開 skills。另有 plugin、marketplace、hooks 目錄。

## 4. Context 管理
- `context/condenser/`：`llm_summarizing_condenser`、`no_op_condenser`、`pipeline_condenser`（可串接）。
- LLM summarizing：把前半事件換成單一摘要，舊摘要可再被摘要；依週期與 `max_tokens`/`max_size` 觸發，也可由 agent/使用者顯式觸發。soft 觸發若無法安全套用就延後；hard 觸發（如 context window error）退回整個 view 的 hard reset。Anthropic tool-use 歷史損毀也用 condensation 修復。
- Repo 層：skills 與 `AGENTS.md`（本 repo 自身使用）；舊 microagents/repo.md 於 V1 的對應細節未驗證。
- 持久化：events 落地；settings 有 `schema_version`（目前 8）、migration 與 golden fixture、CI 檢查。

## 5. 權限與安全
- 隔離：預設 npm 安裝**無沙盒**，agent 有完整檔案系統權限（README 明寫）；Docker 為沙盒選項；本機 launcher 預設綁 loopback。
- `ConversationSettings` 有 `confirmation_mode` 與 `security_analyzer`。Policy：`AlwaysConfirm`、`NeverConfirm`、`ConfirmRisky`（threshold 預設 HIGH，`confirm_unknown` 預設 True）。
- Security 目錄：LLM analyzer、ensemble、shell AST/parser、`defense_in_depth/`、`grayswan/`、toolshield。CLI：預設逐項確認、`--always-approve/--yolo`、`--llm-approve`（LLM 風險評估）。
- 網路出口、secret 隔離的完整模型未驗證（有 `secret` 模組，細節未讀）。

## 6. 復原機制
未驗證存在 checkpoint/rewind/undo 類功能。可確認的只有：append-only event log、可 interrupt/pause、per-conversation git worktree。file_editor 是否保留 undo 歷史（V0 有 undo_edit）在 V1 **未驗證**。

## 7. 模型與成本
- `llm.py` 直接 `import litellm`（completion/responses 同步與非同步），故 provider 廣泛（verified 使用 litellm）。
- 成本：`input_cost_per_token`/`output_cost_per_token`、`Metrics`、`llm.metrics.accumulated_cost`。**未見 `max_budget` 欄位**（只讀到檔案前 100000 字元，後段未讀；故「無硬性預算上限」僅為部分驗證）。
- 另有 retry（預設 5 次）、`fallback_strategy`、`router/`、prompt caching（預設開，retention 24h）、`reasoning_effort` 預設 high、extended thinking budget 預設 200k。
- Hosted：OpenHands Cloud（商業），計價未驗證。

## 8. 上手體驗（推論為主）
- npm 路線：Node 24+ 與 `uv`，一行 `agent-canvas`，但無沙盒；Docker 路線較安全但需 Docker、掛載專案目錄；Windows 另有文件。對新手：需自備 LLM API key（推論）、需理解沙盒/無沙盒取捨、安裝前置條件多。
- 產品線快速變動：V0 → V1、CLI 停維護、主 repo 轉為 Agent Canvas，文件與教學容易過期（推論，依 CLI README 與 issue #15583「CLI parity」）。
- Cloud 為最低門檻選項，但其體驗未實測。

## 9. 擴充與自動化
- Headless：舊 CLI 有 `--headless -t`、`acp`（Zed/VSCode/JetBrains）、`web`、`serve`。
- SDK：Python、TypeScript、REST/WebSocket；examples 含 GitHub workflows。GitHub/GitLab resolver 在目前樹中的位置未驗證。
- Automation：獨立 repo（排程、webhook）。
- **評測（重點）**：OpenHands/benchmarks repo，支援 SWE-Bench、SWE-Bench Pro、GAIA、Commit0、OpenAgentSafety、ProgramBench。兩種 workspace：本機 Docker（按需建 image）與 Remote（runtime API，預設 runtime.eval.all-hands.dev，需 `RUNTIME_API_KEY`，agent-server image 以 SDK commit SHA 標記、由 PR label 觸發建置）。SDK README 標示 SWE-bench 77.6（連結至試算表，具體設定未驗證）；技術報告 arXiv 2511.03690（未能讀取）。已知痛點：benchmark 與 SDK 版本相依（需特定 SDK commit）。

## 10. 優缺點與痛點
優點：MIT、SDK/Server/UI 清楚分層、事件日誌 + 非破壞性 condensation、多種 workspace、風險分級確認、評測基礎建設完整。
缺點：產品面碎片化（Canvas / Agent Server / Automation / SDK / 停維護 CLI）；預設路徑無沙盒；新手概念負擔高。
Issue 佐證（開放 issue 依反應排序，僅能看到留言數）：#7752 多 repo 選擇、#15529 Canvas 更新機制、#15583 slash 指令與 CLI 對齊、#17251 偵測 context/skills 衝突、#16905 ACP 下 Claude Code 看不到內建 skills、#15758 本機模式開啟 repo、#14525 OpenAI 相容 endpoint。
- 新手痛點（推論自上述）：安裝/模式選擇、更新、LLM endpoint 設定。
- 專業痛點：多 repo、skills 衝突、整合一致性、版本相依。

## 11. 對我們的啟發
- 值得抄：append-only event log + 非破壞性 condensation（View 投影）；SDK / agent-server / UI 以 OpenAPI 契約分離；評測 harness 與 remote workspace 並以 commit SHA 綁定 image；風險分級確認（policy × analyzer 可組合）；settings schema versioning + golden fixture + CI。
- 陷阱：別讓預設路徑無沙盒；避免產品線分裂與 CLI 被棄；對新手提供單一入口與預設安全；預算上限應原生支援（OpenHands 似乎缺，未完全驗證）；加入 undo/checkpoint（OpenHands 未證實有）。

## 12. 來源與未能驗證
來源：
- https://github.com/OpenHands/OpenHands 及 README.md
- https://github.com/OpenHands/software-agent-sdk（README、AGENTS.md、sdk/context/condenser、sdk/security、sdk/llm/llm.py、openhands-tools、openhands-workspace 目錄）
- https://github.com/OpenHands/OpenHands-CLI
- https://github.com/OpenHands/benchmarks
- 主 repo issues 列表

未能驗證：docs 站內容、arXiv 論文、SWE-bench 77.6 的設定、Cloud/Enterprise 價格與授權細節、舊 enterprise 目錄現況、CodeAct 在 V1 的角色、IPython 工具、microagents/repo.md 對應、復原機制、max_budget（llm.py 後 41k 字元未讀）、GitHub/GitLab resolver 現況、sandbox 網路/secret 模型、issue 反應數（頁面只顯示留言數）。
