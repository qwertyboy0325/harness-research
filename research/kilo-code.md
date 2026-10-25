# Kilo Code 研究報告

查證日期:2026-10-25。方法:WebFetch 讀 github.com / raw.githubusercontent.com(README、AGENTS.md、`packages/kilo-docs/pages/*.md`、Releases)。注意:kilo.ai / kilocode.ai 網域在此沙箱 DNS 解析失敗,官方 docs 網站本身未直接讀取,改讀 repo 內 docs 原始 Markdown(等同官方文件來源)。「推論」會明確標示。

## 1. 基本資料
- Canonical repo:`github.com/Kilo-Org/kilocode`。License:**MIT**(README/頁面確認)。
- 規模(2026-10-25 頁面):約 27.7k stars、3.3k forks、570 open issues、196 PRs、33,816 commits。最新 release **v7.8.17(10/23)**;10 月內有 v7.8.13~7.8.17 與 JetBrains 7.1.11/7.1.12,節奏約每數日一版(活躍度極高)。
- 技術棧:TypeScript / Bun / Turbo monorepo;CLI 引擎(`packages/opencode`,npm 套件 `@kilocode/cli`)、SolidJS UI(`kilo-ui`)、VS Code extension(`kilo-vscode`)、JetBrains(`kilo-jetbrains`)、`kilo-gateway`、`kilo-indexing`、`kilo-memory`、`kilo-sandbox`、`kilo-telemetry`、`sdk`(`@kilocode/sdk`)、`server`、`tui`。(語言比例頁面未顯示,unverified。)
- 公司/商業模式:由 Kilo-Org 營運,README 宣稱「open source with open pricing」,模型用量依 provider 價格「zero markup」;購買 credits 另收 5% payment-processing fee(gateway 文件);另有 Kilo Pass(文件提及,價格 unverified)、組織方案(共享 credit pool、每人每日上限、模型封鎖清單、用量 dashboard)、Cloud Agent(app.kilo.ai/cloud)、Code Reviews(app.kilo.ai/code-reviews)、managed indexing(官網宣稱,細節 unverified)。收入來源推論:credits 手續費、訂閱與企業功能;具體營收 unverified。

## 2. 架構與血統
**已驗證**:
- Roo Code(Apache-2.0)repo 於 **2026-05-15 被 owner archived**,README 稱「Roo Code Extension was shut down on May 15th」,導向社群 fork ZooCode 與 Cline;24.3k stars。
- Cline(`cline/cline`)**仍活躍**:Apache-2.0、70.8k stars、7,691 commits,描述為「SDK, IDE extension, or CLI」,monorepo 含 `sdk/`(`@cline/sdk`)與 `apps/cli`,稱同一引擎驅動 CLI/桌面/VS Code/JetBrains。
- Kilo 現行版本**不是** Cline/Roo 的 fork:AGENTS.md 明言 Kilo CLI 是 **opencode(anomalyco/opencode)的 fork**;除路徑含 "kilo" 的目錄外全是 upstream 程式碼;Kilo 改動集中在 `packages/opencode/src/kilocode/`,對共享檔案的修改用 `kilocode_change` 標記並由 CI 檢查,定期 merge upstream。
- 舊的 Roo/Cline 血統碼庫已移至 `Kilo-Org/kilocode-legacy`:**2026-08-03 archived**,舊 VS Code/JetBrains 外掛 **2026-07-31 EOL**(無更新與安全修補)。該 README 僅在 callout 提及 Roo 遷移指南,未描述血統。
- 第三方(aiwiki 等,次要來源,unverified):Kilo 2025-03 起自 Roo fork,2026-02 重建於 opencode。我只能由官方資料確認「現行=opencode fork、legacy 已封存」,時間點細節未驗證。
**多 surface 架構(AGENTS.md)**:核心是 `kilo serve` 的 HTTP+SSE server;VS Code 外掛內嵌 CLI binary 並以子行程啟動它;sidebar、編輯器 tab、Agent Manager 共用同一連線;SDK 由 server API 自動生成;JetBrains 另有 package(有獨立 RELEASING.md,實作細節 unverified);另有 Zed extension、mobile app、Slack/Linear/GitHub 整合頁。此「單一 headless 引擎 + 多前端」是最值得學的架構。

## 3. 工具集與 modes
- 內建 tools(docs):`read`、`glob`、`grep`、`edit`、`write`、`apply_patch`、`bash`、`webfetch`、`websearch`(Kilo/OpenRouter 用戶)、`question`、`task`(開 sub-agent)、`todowrite/todoread`、`plan`、`skill`;另有 `agent_manager` tool、`kilo_memory_save/recall`、browser 工具(`browser_open` 權限出現在 release notes)。
- Agents:`code`(預設)、`ask`(唯讀)、`plan`(唯讀+只能寫 `.kilo/plans/`)、`debug`、`orchestrator`(**已 deprecated,將移除**)。注意舊版「Architect」在新版叫 Plan。自訂 agent 以 Markdown+YAML frontmatter(`permission` 鍵)定義,另有 custom subagents、skills、workflows 文件。
- Sub-task:Code/Plan/Debug 可直接用 `task` 開 subagent(前景/背景、可並行),subagent 共用父層目錄不隔離檔案;`task` 預設權限 `ask`;「Kilo Swarm」預設開啟,讓 session 與子任務共享 findings board(`shared_agent_board`)。隔離分支建議用 Agent Manager worktree。
- Agent Manager(VS Code):多 session、每個可用獨立 git worktree(`.kilo/worktrees/`),multi-version 模式最多 4 個平行實作,diff 面板與 PR 整合(靠 `gh`)。
- MCP:docs 有 `automate/mcp/` 六頁(overview、transports、CLI/extension 使用);MCP 也可經 Marketplace 安裝,OAuth 有支援(release notes)。tools 頁本身未提 MCP。

## 4. Context 管理
- Rules:`.kilo/rules/`(專案)、`~/.kilo/rules/`(全域),mode 專屬資料夾如 `.kilo/rules-code/`(出自 migrating 文件,新舊版是否完全一致 unverified);`AGENTS.md` 自動載入(`AGENT.md` fallback),且為寫保護需核准;`.kilocodeignore` 與 deny 規則排除路徑。
- Memory:舊 **memory bank 已 deprecated**,由 AGENTS.md 取代。新「Kilo Memory」為 opt-in、僅專案層級;`project.md`/`environment.md`/`corrections.md` + session digests,存於 repo 外 `~/.local/share/kilo/memory/...`;自動存取(只增改不刪、過濾疑似 secret),`/memory remember|correct|forget|purge`;記憶優先權低於當前訊息、repo 檔案與 AGENTS.md。
- Compaction:預設開;以 anchored summary 取代舊歷史,保留最近 `tail_turns=2`;另有 prune(超出 40K token 的 tool output 換成 placeholder);可設 `threshold_percent`、專用 compaction model;`/compact`。
- Codebase indexing:**預設關閉**;embedding 支援 OpenAI/Gemini/Mistral/Voyage/Bedrock/OpenRouter/Ollama/OpenAI-compatible;向量庫預設 LanceDB(本機嵌入式),可選 Qdrant;tree-sitter 本地解析;可完全離線(Ollama+LanceDB)。

## 5. 權限與安全
- 三態 `allow / ask / deny`,以 `kilo.jsonc` 全域設定或 agent frontmatter 覆寫;可到單一 tool 與 glob pattern(`bash` 的 `git *`、`edit` 的 `src/*`);規則「最後匹配者勝」;shell 命令逐段解析,任一被拒則整體拒絕;`external_directory` 另控;`.env`/`.env.*` 讀取即使廣泛 allow 也會 ask(`.env.example` 預設允許);`doom_loop`(重複失敗)預設 `ask`;VS Code 有 shield 按鈕即時切換 auto-approve;CLI per-agent `steps` 上限。
- `kilo run --auto` 會自動核准一切(除非被 deny 規則攔),README 警告僅限可信環境。非互動且無 `--auto` 則自動拒絕並以 exit 1 結束。
- Marketplace 文件明說「validation is not a security review」,plugins 以完整權限執行。
- 近期 release 有安全修補(MCP OAuth URL 限 HTTP(S)、MCP prompt 的 shell expression 視為純文字),顯示 MCP/供應鏈面向有真實風險。
- 無 OS 級 sandbox 的確切機制:repo 有 `kilo-sandbox` package,但用途 unverified。

## 6. 復原機制
Checkpoints 為 git-based snapshot,新版使用 repo 外的獨立 snapshot repo(不動使用者 `.git`),在每次 model call 前後記錄;遵守 `.gitignore`。VS Code 於 user message 上「Revert to here」,有 Redo/Redo All,再送新訊息才永久刪除;CLI 支援整體 revert、undo revert、per-file revert。限制:粒度為每個 user message;只涵蓋 Kilo 做的變更;需安裝 git;每小時 `git gc` 清除 7 天以上不可達物件(舊 snapshot 可能遺失);還原會覆蓋未存檔變更。

## 7. 模型與成本
- Kilo Gateway(`kilo-gateway` package + `gateway/` 文件有 API reference、SDK、streaming):帳號可用 500+ 模型,按上游價格扣款,內部以 microdollars 計費;餘額為 0 時付費模型回 HTTP 402;BYOK 時 Kilo 成本記 $0;支援 30+ providers,也可用 ChatGPT 訂閱(Go/Plus/Pro)登入使用。
- 「無需 API key」:README 稱 "No API keys required to start";`kilo-auto/free`(Auto Free)自動路由到可用免費模型,免費模型標示 "(free)",限 **200 requests/hour/IP**;文件警告 free 端點(含 NVIDIA)可能記錄 prompt 與輸出,勿送機密。免費模型清單會變動(具體名單 unverified)。免費是否需帳號:docs 未明說;setup 頁則提示「sign in or create a free account」。是否有贈送 credits:文件未提(unverified)。
- 也有 `kilo-auto/efficient`、自訂 `kilo-auto/custom/<slug>` auto routers(`kilo auto router/eval` 指令)。背景任務(標題、摘要)預設用 Auto Small 會耗 credits,可改免費 small_model。
- 成本控制:餘額上限、組織每人每日上限(UTC 午夜重置)、spend alert(滾動 24h/7d/30d 與 spike 偵測,email/mobile push)、auto-compaction、prompt caching、輸出 token 建議 ≤16k。是否在對話中逐 turn 顯示費用:未於所讀文件確認(unverified)。

## 8. 上手體驗
安裝:VS Code Marketplace、JetBrains Marketplace、CLI(npm/curl/pnpm/bun/Homebrew/AUR/GitHub Releases binary,含 musl 與非 AVX 版)。首次執行:VS Code 提示 sign in(email、Google、GitHub、Apple 等多種 OAuth),瀏覽器完成後即有預設模型可用,再用自然語言下任務;多數 tool 自動執行,但 shell、外部目錄、敏感檔讀取仍詢問。CLI 需 `kilo auth login`。
降低門檻處:免 API key、一鍵 OAuth、Auto 模型路由(使用者不必選模型)、`/init`、Cursor/Windsurf 規則遷移文件、Cloud Agent 免本機環境、mobile/Slack 入口。
仍然困難(推論,基於文件結構):概念多(agents、subagents、skills、plugins、MCP、rules、AGENTS.md、`kilo.jsonc`、worktree、permission glob 規則);checkpoints 需 git;Agent Manager 需 git repo 與 `gh`;配置檔(JSONC)與權限 pattern 對新手不友善;免費額度 200 req/h 與隱私警告。

## 9. 擴充與自動化
CLI:`kilo run "<task>" --auto`、`--format json`(輸出 schema 文件未詳述)、exit code 0/124/1、`--continue`、`KILO_ORG_ID`、OpenTelemetry 預設開啟(可關)、`/resume-claude`、`/resume-codex` 匯入其他 CLI 逐字稿;`kilo serve` + `@kilocode/sdk`。整合:GitHub、Slack、Linear、Cloud Agent、Code Reviews(auto-triage 文件存在)、mobile 遠端控制 CLI。Marketplace:Agents/Skills/MCP servers/Plugins,可裝 project(`.kilo/`)或 global,安裝前顯示將寫入的檔案。

## 10. 優點 / 缺點 / 痛點
優點:真開源 MIT 且高頻發布;單一 server 引擎多前端;細緻 permission 模型;worktree 平行 agent;全面的免費/BYOK 路徑;記憶與 compaction 可設定;完整遷移文件。
缺點(部分為推論):
- 血統劇烈變動:2026-02 前後重建、legacy 外掛 2026-07-31 EOL,舊用戶被迫遷移;Orchestrator、memory bank 已棄用,功能名稱快速變動,文件與第三方教學易過時。
- 跟 opencode upstream 持續 merge,維護成本高(AGENTS.md 為此訂了嚴格規範)。
- 範圍膨脹(autocomplete、cloud、mobile、gateway、reviews),核心易被稀釋。
Issue 佐證(僅標題層級,反應數未顯示):#8108「Why was this large 'upgrade' introduced?」、#5460「Gemini CLI removed」、#822/#7253 multi-root workspace 支援、#3058/#4498 autocomplete 自訂 provider、#3679 Shift+Enter 不換行;皆為已關閉。表示使用者對大版本變更、功能移除與工作區支援有反應。深入的留言內容未讀,不下結論。
新手痛點(推論):設定項過多、auto-approve 風險難理解、免費模型品質與限速、費用可預測性。
專業者痛點(推論):權限規則「最後匹配勝」易寫錯、subagent 不隔離檔案、snapshot 7 天清理、plugin 供應鏈風險、與 upstream 分歧。

## 11. 對我們的啟發
可複製:
1. 單一 headless server + SDK,多前端(IDE/CLI/Web/mobile)共用;
2. 預設可用的 auto 模型路由與「免 key」首跑體驗;
3. repo 外的 snapshot repo 做 checkpoint,不污染使用者 git;
4. 權限三態 + 敏感檔強制 ask + doom-loop 保護 + 一鍵 shield 切換;
5. Agent 以 Markdown+frontmatter 定義,可進版控;
6. 安裝 Marketplace 項目前顯示將改動的檔案;
7. 從競品(Cursor/Windsurf)的規則遷移文件。
避免:
1. fork 大型 upstream 的長期 merge 負擔(Kilo 靠 `kilocode_change` 標記與 CI 緩解);
2. 功能名稱頻繁更換(modes→agents、memory bank→AGENTS.md→Kilo Memory);
3. 預設過寬的自動核准。
商業模式張力(推論):gateway 是營收入口,而開源客戶端必須同時支援 BYOK/本地模型;免費路由會讓使用者資料流向第三方 provider(Kilo 自己有警語),預設導向 gateway 的 UX 會與「開源中立」產生矛盾;Roo 在 2026-05 關閉顯示純開源擴充難以獨立維持。建議:gateway 與 client 協議開放、預設不綁定、隱私揭露放在首跑畫面。

## 12. 來源與未能驗證
來源(皆 2026-10-25 讀取):
- https://github.com/Kilo-Org/kilocode(README、/releases、/tree/main、/packages)
- https://raw.githubusercontent.com/Kilo-Org/kilocode/main/README.md、AGENTS.md
- https://raw.githubusercontent.com/Kilo-Org/kilocode/main/packages/kilo-docs/pages/ 下:customize/agent-permissions.md、customize/marketplace.md、customize/context/{codebase-indexing,memory,context-condensing}.md、getting-started/{quickstart,setup-authentication,using-kilo-for-free,cost-controls-and-usage-safeguards,migrating}.md、gateway/usage-and-billing.md、automate/{how-tools-work,agent-manager}.md、code-with-ai/agents/{using-agents,orchestrator-mode}.md、code-with-ai/features/checkpoints.md、code-with-ai/platforms/cli.md
- https://github.com/Kilo-Org/kilocode-legacy、https://github.com/RooCodeInc/Roo-Code、https://github.com/cline/cline
- 次要(僅用於標示血統時間線,未採為事實):WebSearch 結果之 aiwiki.ai、morphllm.com 比較文。
未能驗證:
- kilo.ai / kilocode.ai 官網(DNS 失敗),故 Kilo Pass 價格、managed indexing 細節、官網宣稱數字皆 unverified;
- 使用者實際免費額度/是否免帳號、是否逐 turn 顯示費用;
- 2025-03 自 Roo fork、2026-02 重建的確切日期(僅第三方);
- Cline 近期 SDK 重寫細節與 release 版本;Roo 最後 release;
- `kilo-sandbox`、JetBrains 實作細節;`--format json` schema;
- 各 issue 的內文與反應數;語言佔比與下載量/用戶數。
