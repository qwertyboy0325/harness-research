# OpenCode 研究報告

檢查日期：2026-10-25。標記：【已驗證】= 來自一手來源（GitHub repo / raw 文件）；【推論】= 本人推測。注意：opencode.ai 網域在本環境無法連線（DNS 失敗），故所有文件均取自 repo 內 `packages/web/src/content/docs/*.mdx`（官方文件原始檔），README 引用的 opencode.ai/v2/docs 未能直接讀取。

## 1. 基本資料
- 【已驗證】Canonical repo：github.com/anomalyco/opencode（org: anomalyco；舊名 sst/opencode，GitHub 載入 sst/opencode 顯示為 anomalyco/opencode）。
- License：MIT。Stars 約 221.4k、forks 約 29.6k（2026-10-25 頁面顯示）。
- 技術棧（AGENTS.md）：TypeScript + Bun、Effect（services / schema）、Drizzle + SQLite、OpenTUI（TUI）、Playwright / Storybook 測試。monorepo 含 35+ packages（cli、core、server、client、sdk、protocol、schema、tui、app、desktop、web、plugin、console、containers、codemode、ai、session-ui、theme…）。
- 發佈節奏：近 8 個 release（v1.18.33 於 9/28 → v1.19.4 於 10/22）約每 3.5 天一版；最新 v1.19.4（10/22）。v1.19.0 加入 `opencode upgrade --major`（通往 OpenCode 2），1.x 不再自動更新；v1.19.1 終止 session sharing（含既有分享連結）。目前處於 1.x → 2.x 過渡期（README 與 v2 文件並存，兩版共用 `opencode` 指令會衝突）。
- 商業模式：【已驗證】提供 OpenCode Zen（經團隊測試的精選模型 gateway，需儲值計費）與 Go（低價訂閱開源 coding 模型），兩者皆為選用。融資/股權背景：未驗證（未讀到一手來源）。

## 2. 架構
- 【已驗證】Client/Server 分離：`opencode serve` 啟動 HTTP server（預設 port 4096），TUI / desktop / web / IDE / SDK 皆為其 client；SDK 型別由 server 的 OpenAPI/HttpApi 產生（`packages/client` 為 generated，禁止手改）。
- 分層（AGENTS.md）：Schema → Core、Protocol → Server；client runtime 只能依賴 Schema 與 Protocol。
- Session core 設計（AGENTS.md，屬 v2 重構）：durable event 只記錄不可還原的新事實；`session.prompt` 先寫入 durable inbox 再喚醒執行；執行為 process-global、以 Session ID 為 key，runner/model/tools/permissions/檔案存取以 Location 為範圍；每一 step 是一次明確的 `llm.stream` 呼叫並重新載入歷史，「沒有記憶體內的 tool loop」；prompt 預設為 steer，steered compaction 優先。
- 儲存：Drizzle + SQLite（AGENTS.md）；憑證在 `~/.local/share/opencode/auth.json`。session 資料確切路徑/schema：未驗證。
- UI：TUI（預設）、Desktop app（beta，macOS/Win/Linux）、web、ACP（編輯器協定）、IDE 整合（docs 有 ide.mdx、acp.mdx）。

## 3. 工具集
【已驗證，tools.mdx】
- `bash`、`edit`（精確字串取代，主要修改方式）、`write`（建立/覆寫）、`apply_patch`（`*** Add/Update/Move/Delete File:` 風格 patch；與 edit/write 共用 `edit` 權限 key）、`read`（可指定行範圍）、`grep`（regex）、`glob`、`lsp`（實驗性，需 `OPENCODE_EXPERIMENTAL_LSP_TOOL=true`；definition/references/hover/symbols/call hierarchy）、`skill`（載入 SKILL.md）、`todowrite`、`webfetch`、`websearch`（Exa 或 Parallel，免 API key）、`question`（向使用者提問）。
- 編輯後可串 formatter（`formatter` 設定，預設關閉，`true` 啟用內建）；LSP servers 另有設定（lsp.mdx），診斷回饋給模型的細節：未驗證。
- MCP：支援，權限可用萬用字元如 `"mymcp_*": "ask"`。
- 自訂工具：config 目錄中定義（custom-tools.mdx）；plugin 內以 `tool()` helper + Zod schema + `execute(args, context)`；同名 plugin tool 覆蓋內建工具。

## 4. Context 管理
- 規則檔：由 cwd 向上找 `AGENTS.md`，其次 `CLAUDE.md`；全域 `~/.config/opencode/AGENTS.md`，再 fallback `~/.claude/CLAUDE.md`；`~/.claude/skills/` 也相容；`instructions` 欄位可列檔案、glob、遠端 URL（5 秒 timeout）。`/init` 會分析專案並產生 AGENTS.md。
- 壓縮：`compaction.auto`（預設 true）、`compaction.prune`（預設 false，移除舊 tool output）、`reserved` buffer；`/compact`（別名 `/summarize`）手動；plugin hook `experimental.session.compacting` 可注入或取代 compaction prompt；有內建隱藏的 compaction/title/summary system agents。
- 引用：`@` 模糊檔案搜尋並注入內容；`!` 前綴執行 shell，輸出入對話。
- Agents：primary 為 Build（預設全權限）與 Plan（拒絕編輯，僅允許 plan 目錄的 Markdown，bash 依權限）；Tab 切換。Subagent：General（可並行、可編輯、無 todowrite）、Explore（唯讀快速搜尋）；`@` 手動呼叫或由 Task tool 自動委派。自訂 agent：JSON 或 `.opencode/agents/*.md` frontmatter（description、mode、model、prompt、temperature、steps、permission、hidden…）；`permission.task` 控制可呼叫的 subagent。
- 系統 prompt 細節與 repo map / 索引策略：未驗證（未讀 prompt 原始碼；從文件看無 embedding 索引，靠 grep/glob/read 與 LSP——【推論】）。

## 5. 權限與安全
- 【已驗證，permissions.mdx】三值 `allow / ask / deny`；可設 `*` 全域預設，並按工具、按 bash 指令 pattern（`*`、`?` 萬用字元）設定；**最後匹配規則勝出**。多數工具預設 `allow`；`doom_loop`（同一 tool call 相同輸入重複 3 次）與 `external_directory`（存取啟動目錄外路徑）預設 `ask`；`.env`、`.env.*` 讀取需 ask，`.env.example` 允許。ask 的回覆：once / always（本 session）/ reject。per-agent 權限與全域合併，agent 優先。`--auto` 自動核准會 prompt 的請求但不覆蓋 explicit deny。
- 【已驗證，SECURITY.md】官方明言：**權限系統不是 sandbox、不是安全邊界**，僅是 UX 功能；沙箱逃逸不在漏洞範圍；建議用 Docker/VM 隔離。repo 有 `packages/containers`，但內建 sandbox 執行：未驗證（文件中未見）。
- 實際 issue 例（標題取自搜尋頁，狀態以頁面為準）：#58638 symlink 可繞過 read 規則與 external_directory 檢查（open）；#58537 Plan mode 可經 MCP tools 修改檔案（closed）。顯示「預設多數 allow」與規則比對的弱點。

## 6. 復原機制
- 【已驗證】`/undo`（ctrl+x u）移除最近一則使用者訊息、其後所有回應與該輪檔案變更，並把原訊息放回輸入框供改寫，可連續執行；`/redo`（ctrl+x r）重做。文件稱「使用 Git 追蹤檔案變更，專案必須是 Git repo」。`snapshot` 設定預設啟用，關閉後 UI 無法回滾。
- 限制（【已驗證】僅標題層級）：snapshot 在大型 untracked 樹/ Windows $HOME 會卡住（#57841，重複 issue）；work tree 有 unix socket 時 snapshot 失敗（#58371）；生成中途 revert 只還原 prompt 而未還原檔案（#57383）；被拒絕的 compaction 會截斷歷史並清除暫存 undo（#57553）。
- 只還原 agent 的檔案變更；bash 副作用（安裝套件、資料庫、repo 外檔案）【推論】無法還原。是否使用 shadow git repo 或實際 commit：未驗證。

## 7. 模型與成本
- 【已驗證】透過 AI SDK + Models.dev 支援 75+ providers；`/connect` 存 key；可設 `baseURL`、`whitelist/blacklist` 模型。本地模型：任何 OpenAI-compatible 端點（Ollama、LM Studio、llama.cpp、Atomic Chat）以 `@ai-sdk/openai-compatible` 設定；Ollama 工具呼叫失敗時建議 `num_ctx` 調至 16k–32k。
- 登入：GitHub Copilot device flow、OpenAI「ChatGPT Plus/Pro」瀏覽器登入。Anthropic 訂閱（Claude Pro/Max）：文件稱 Anthropic 明確禁止，1.3.0 起不再內建此類 plugin（#7410「Broken Claude Max」為 repo 內留言最多的 issue 之一，386 則）。
- 成本顯示/預算上限：文件未見預算控制；有 context 用量需求 issue #6152（已關閉）、Go 用量 API #16017。TUI 具體成本顯示：未驗證。Agent 層有 `steps` 上限可限制迭代。

## 8. 上手體驗
- 安裝：`curl -fsSL https://opencode.ai/install | bash`、`npm i -g opencode-ai`、bun/pnpm/yarn、Homebrew、Arch、mise、Nix；Windows 建議 WSL；另有 desktop app（beta）。
- 首次：進入專案執行 `opencode` → `/connect` 選 provider 貼 key（新手推薦 Zen）→ `/init` 產生 AGENTS.md → 對話。必設：provider/model；可選：theme、keybinds、formatter、commands。
- 痛點（【推論】基於文件）：要自行理解 provider/key、Build vs Plan、Git 才有 undo；1.x/2.x 並存的安裝衝突。

## 9. 擴充與自動化
- Plugins：`.opencode/plugins/` 或 `~/.config/opencode/plugins/` 的 JS/TS，或 config `plugin` 陣列列 npm 套件（Bun 安裝）。事件/hook 包含 `tool.execute.before/after`、`shell.env`、`permission.asked/replied`、`session.*`（created/idle/compacted/diff/error）、`file.edited`、`lsp.client.diagnostics`、`message.*`、`todo.updated`、`tui.*`。
- 其他：自訂 commands、skills、themes、keybinds、formatters、config 多層合併（remote well-known → 全域 → 專案 → .opencode → 環境變數 inline → managed/企業）；enterprise/policies 文件存在（細節未讀）。
- 無頭/CI：`opencode serve`、`opencode run`（存在於 CLI，【推論】；sdk.mdx 未記載細節）。SDK：`@opencode-ai/sdk`，`createOpencode()`/`createOpencodeClient()`，v2 支援 JSON schema structured output。
- GitHub：`opencode github install` 建立 app + workflow；留言 `/opencode` 或 `/oc`；支援 issues、pull_request（預設審查）、schedule、workflow_dispatch；OIDC 或 `use_github_token`。GitLab 亦有文件。

## 10. 優缺點與痛點
優點：真正 client/server、多 UI；provider 無關且支援本地；規則/skills 相容 Claude Code；細緻 pattern 權限；plugin 事件豐富；MIT 且活躍。
缺點：安全模型薄弱（預設多數 allow、非 sandbox、symlink 繞過）；undo 依賴 Git snapshot，邊界案例多；1.x→2 重構期的相容性混亂；release 極頻繁（穩定性風險，【推論】）；官方已終止 session sharing。
新手痛點：選 provider/API key、成本不透明、不知道 undo 的限制、需要 Git。專業痛點：權限規則「最後勝出」難推理、大 repo snapshot 卡住（#57841）、Plan mode 經 MCP 可寫檔（#58537）、剪貼簿問題（#4283，157 則留言）、編輯器整合需求（#11176 官方 VS Code 擴充 open）、/goal 長任務需求（#27167，84）。

## 11. 對我們的啟發
可複製：(1) server 為核心、多 client；(2) AGENTS.md 並相容 CLAUDE.md 與 skills，降低遷移成本；(3) `/init`、`/undo`/`/redo`（並還原使用者訊息）的互動；(4) 精選模型 gateway（Zen）給新手，免選型；(5) Plan/Build 雙模式；(6) doom_loop 偵測；(7) 事件型 plugin hook；(8) `.env` 預設需確認。
避免：(1) 預設全 allow——新手應預設 ask 或內建隔離（容器/worktree）；(2) 只靠 Git snapshot 且要求 Git repo——應自動初始化並在 UI 說明 bash 副作用不可復原；(3) 權限被當作安全邊界卻不是——要明確標示或真的做 sandbox；(4) symlink / MCP 繞過，權限需在路徑解析後檢查、涵蓋 MCP 工具；(5) 缺少預算/成本上限；(6) 大版本並存的安裝衝突；(7) 頻繁發版缺少穩定通道。

## 12. 來源與未能驗證
已讀 URL：
- https://github.com/sst/opencode（顯示為 anomalyco/opencode）
- https://raw.githubusercontent.com/anomalyco/opencode/dev/AGENTS.md
- https://raw.githubusercontent.com/anomalyco/opencode/dev/SECURITY.md
- https://github.com/anomalyco/opencode/releases
- https://github.com/anomalyco/opencode/tree/dev/packages 及 .../packages/web/src/content/docs
- raw .../packages/web/src/content/docs/{tools,permissions,agents,index,tui,plugins,config,providers,github,rules,sdk}.mdx
- GitHub issues 搜尋頁（reactions 排序、undo/snapshot、sandbox 關鍵字）；僅讀到清單標題，未開啟個別 issue 內文。

未能驗證：opencode.ai 網站本身（DNS 失敗）；融資/公司背景；系統 prompt 內容；repo map/索引策略；LSP 診斷回饋模型的方式；snapshot 是否為 shadow git repo；session SQLite schema；TUI 成本/token 顯示與預算控制；`opencode run/serve` 參數；內建 sandbox；sessions 分享功能舊行為；contributors 數與語言比例；README 所稱 v2 文件內容；issue 搜尋結果中的 reproduced/severity 標註為 WebFetch 摘要，未逐一核對。
