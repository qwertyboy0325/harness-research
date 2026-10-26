# 開源 Harness 跨工具比較

查證日期：2026-10-25。資料全部來自 `research/` 下 8 份個別報告（opencode、cline、kilo-code、aider、codex-cli、goose、openhands、hermes-agent）。§1–§6 的表格只涵蓋前 7 家，**Hermes Agent 的對照放在 §10**（定位不同，單獨比較較清楚）；§3 的觀察已把它納入。
**可信度說明：** 各家官方文件站在查證環境都連不上（DNS／proxy），報告改讀 repo 內的 markdown 與原始碼；issue 多數只讀到標題或搜尋摘要。star 數與版本號變動快，僅供參考。「未驗證」表示該報告沒能從一手來源確認，不代表不存在。

## 0. 對先前口頭說法的更正

| 先前說法 | 查證後 |
|---|---|
| Kilo Code 是 Cline／Roo 系分支 | 現行 Kilo 是 **opencode 的 fork**（官方 AGENTS.md 明載）。舊的 Roo/Cline 血統碼庫已移到 `kilocode-legacy`，2026-08 封存，舊外掛 2026-07-31 EOL。Roo Code 本身 2026-05-15 封存 |
| OpenHands 以 Docker sandbox 隔離為特色 | 預設的 npm 安裝路徑**沒有 sandbox**，agent 有完整檔案系統權限。Docker 只是選項 |
| Aider 是成熟穩定的選擇 | 能見到的最新 release 是 v0.86.0（年份頁面未顯示，推論為 2025 年），約一年多沒新版；單一核心維護者；`main` 仍有零星 commit |
| OpenCode 有 undo | 有，但依賴 Git snapshot 且專案必須是 Git repo，邊界案例多（見 §3） |
| Goose 的 lead/worker 模型設定 | 現行文件找不到，可能已移除 |

## 1. 基本資料

| | OpenCode | Cline | Kilo Code | Aider | Codex CLI | Goose | OpenHands |
|---|---|---|---|---|---|---|---|
| Repo | anomalyco/opencode | cline/cline | Kilo-Org/kilocode | Aider-AI/aider | openai/codex | aaif-goose/goose | OpenHands/OpenHands + software-agent-sdk |
| 授權 | MIT | Apache-2.0 | MIT | Apache-2.0 | Apache-2.0 | Apache-2.0 | MIT |
| 語言 | TS/Bun | TS | TS/Bun（opencode fork） | Python | Rust | Rust | Python SDK + TS UI |
| Stars（約） | 221k | 71k | 28k | 50k | 131k | 56k | 92k（主 repo） |
| 組織 | anomalyco | Cline Bot Inc. | Kilo-Org | 個人（P. Gauthier） | OpenAI | Linux Foundation AAIF | All Hands AI |
| 發版節奏 | 約每 3.5 天 | 每週多版 | 每數日 | 一年以上無新 release | 穩定版約每週、alpha 近每日 | 約每週 | 高 |
| 主要入口 | TUI、桌面、web、IDE、SDK | VS Code／JetBrains、CLI、桌面、SDK | VS Code、JetBrains、CLI、Cloud | 終端 | 終端 TUI、IDE、桌面、Web | 桌面、CLI、API | Agent Canvas（自架 UI）、SDK、雲端 |
| 營利方式 | Zen 模型 gateway、Go 訂閱 | Cline Provider、ClinePass $9.99／月 | Gateway（無加價＋5% 儲值費）、Pass、組織方案 | 無 | OpenAI 帳號／API | 無（基金會；導流 Tetrate） | OpenHands Cloud／Enterprise |

注意：**OpenCode、Cline、Kilo、Goose、OpenHands 都在近期做了大幅重構**（OpenCode 1.x→2.x、Cline v4 SDK 重寫且曾回滾、Kilo 2026-02 重建、Goose v1.55 移除舊 agent loop、OpenHands V0→V1 且 CLI 停維護）。這個領域的架構還沒定型，文件和教學很容易過期。

## 2. 架構對照

| | 核心形態 | 前端共用方式 | agent loop |
|---|---|---|---|
| OpenCode | `opencode serve` HTTP server | 所有 client 連同一 server | 每個 step 是一次明確 `llm.stream` 並重載歷史，v2 設計「沒有記憶體內 tool loop」 |
| Cline | stateless `@cline/agents` ＋ 本機 hub daemon（WebSocket） | CLI／桌面／extension 共用 daemon | stateless loop，狀態在 core |
| Kilo | `kilo serve` HTTP＋SSE | VS Code、JetBrains、SDK 共用 | 繼承 opencode |
| Aider | 單一 Python 聊天迴圈 | 無 | **不是 tool-calling agent**：LLM 輸出文字格式的編輯，由 aider 解析套用 |
| Codex | Rust `core`，SQ/EQ 佇列＋ app-server JSON-RPC | TUI、exec、IDE 皆為 core 前端 | Session → Task → Turn |
| Goose | Rust core，CLI／桌面／API 共用 session DB | 同核心 | v1.55 移除舊 loop（細節未驗證） |
| OpenHands | SDK → tools → workspace → agent-server（REST＋WS）→ UI | OpenAPI 契約 | append-only event log；condensation 用標記事件 |

**共通模式：** 幾乎所有新一代工具都收斂到「一個 headless 核心＋多個前端」。這已經是標準做法，不是差異化。

## 3. 安全與復原（對新手最關鍵）

| | 預設權限 | 真正的隔離 | 檔案復原 | 復原的限制 |
|---|---|---|---|---|
| OpenCode | 多數工具預設 **allow**；`.env` 讀取、外部目錄、重複失敗預設 ask | **無**。SECURITY.md 明言權限系統不是 sandbox | `/undo`／`/redo`（還原使用者訊息） | 依賴 Git snapshot，專案必須是 Git repo；大型樹會卡住（#57841）；bash 副作用推論無法還原 |
| Cline | 檔案編輯與命令預設需核准；但**CLI 預設自動核准** | 無 OS sandbox；是否需核准由模型自己標 `requires_approval`，沒有硬性黑名單 | **Shadow git checkpoint**（每次 tool use 一次，含未追蹤檔，三種還原模式） | 無數量上限／清理策略文件；大 repo 耗儲存；CLI 是否有 checkpoint 未驗證 |
| Kilo | allow／ask／deny 到 tool＋glob，最後匹配勝；`.env` 強制 ask；doom_loop 預設 ask | 無 OS sandbox（`kilo-sandbox` 用途未驗證） | repo 外的獨立 snapshot repo，不動使用者 `.git` | 粒度為每則 user message；需安裝 git；7 天後不可達物件被 gc |
| Aider | 沒有權限層 | 無 | 每次編輯自動 git commit；`/undo` | 只能撤銷 HEAD 且須是本 session 的 aider commit；已 push 就拒絕 |
| Codex | `on-request`＋`workspace-write`＋**預設無網路** | **最強**：macOS Seatbelt、Linux bubblewrap＋seccomp、Windows 三種後端；`.git` 與 `.codex` 唯讀 | **沒有檔案級 undo**（Esc 只回溯對話）；靠 git | Windows sandbox 佈建失敗時只能退到 Full access；`.git` 唯讀導致 worktree 反覆核准 |
| Goose | 預設 **Completely Autonomous**（最寬） | 無 OS sandbox；有 prompt-injection 偵測與 Adversary Mode（fail-open） | **沒有**，文件只建議用 git | 權限只到 tool 層級，shell 命令層級規則仍是 open issue #11399 |
| OpenHands | CLI 逐項確認；有 ConfirmRisky＋多種 analyzer | Docker／apptainer／cloud 可選，但**預設 npm 路徑無 sandbox** | **未驗證存在** | — |

**觀察：**
1. 八家裡只有 Cline、Kilo、Hermes 做了「不依賴使用者 git、不污染使用者 git」的 checkpoint；其中 Cline 較完整，Kilo 有 7 天 gc 的限制，**Hermes 的 checkpoint 預設關閉**且容器 backend 不支援（見 §10）。
2. 只有 Codex 做了真正的 OS 級 sandbox，但它沒有檔案復原，Windows 還常失敗。**沒有任何一家同時做到「強隔離＋可靠復原」。**
3. 預設值偏寬鬆是常態（OpenCode、Goose、Cline CLI）。新手拿到的是最危險的預設。
4. 兩種權限實作的弱點都有實例：OpenCode 的 symlink 繞過（#58638）、Plan mode 經 MCP 可寫檔（#58537）；Cline 把判斷交給模型。

## 4. 成本與模型

| | provider 範圍 | 成本顯示 | 預算上限 | 新手路徑 |
|---|---|---|---|---|
| OpenCode | 75+，本地 OpenAI-compatible | TUI 顯示未驗證 | **無**（只有 agent `steps` 上限） | Zen gateway；`/connect` |
| Cline | 廣，含本地 | 每次呼叫顯示（訂閱計費 provider 隱藏） | **未驗證** | Cline 帳號／ClinePass |
| Kilo | 500+（經 gateway），BYOK | 逐 turn 顯示未驗證 | 餘額上限、組織每人每日上限、spend alert、doom-loop 保護 | **免 API key**，OAuth＋`kilo-auto/free`（200 req／小時） |
| Aider | litellm，廣 | token／cost 回報 | 無 | 需自備 key |
| Codex | 以 OpenAI 為主，自訂 provider 只能講 Responses API | `/status`、`/usage`；金額未驗證 | 無 | ChatGPT 帳號登入 |
| Goose | 15+，本地含內建推論 | 僅 CLI 有 `GOOSE_CLI_SHOW_COST`；桌面未驗證 | 無 | OAuth 一鍵，Tetrate 贈 $10 |
| OpenHands | litellm | 有 `accumulated_cost` | 未見 `max_budget`（只讀到檔案前 100k 字元） | 需自備 key；Cloud |

**觀察：** **Kilo 是唯一有成體系成本控制的**（但綁在它自己的 gateway 上）。其他家要不是沒有上限，就是沒驗證到。免 key 的入口（Kilo、Goose、OpenCode Zen、Cline）全都綁定某個商業 gateway，這就是下面 §7 的利益衝突。

## 5. 上手體驗

| | 新手能否免設定開始 | 主要摩擦 |
|---|---|---|
| OpenCode | 要 `/connect` 選 provider 貼 key（推薦 Zen） | 要懂 provider、Build／Plan、Git 才有 undo；1.x／2.x 並存衝突 |
| Cline | 要先選 provider 取得 key 或登入 | 5 種介面、3 種計費、預設模型數日內換來換去 |
| Kilo | **可以**（OAuth，免 key） | 概念多（agents、subagents、skills、MCP、rules、worktree、JSONC 權限）；免費額度與隱私警告 |
| Aider | 否 | 需終端機、git、API key、手動 `/add` 檔案、Python 環境 |
| Codex | 登入 ChatGPT 即可 | approval×sandbox 兩軸概念；Windows 設定失敗 |
| Goose | **可以**（OAuth） | 預設最寬權限；keyring、M3、Windows 等平台問題 |
| OpenHands | 否 | 需 Node 24＋uv 或 Docker；產品線分散 |

## 6. 共通點（入場門票）

七家**全部都有**：
1. 模型中立（Codex 例外：實質綁 Responses API）。
2. MCP 支援（**Aider 沒有**）。
3. 規則檔：`AGENTS.md` 已成事實標準，OpenCode、Cline、Kilo、Codex 都讀，多數也相容 `CLAUDE.md`／`.cursorrules`。
4. Context 壓縮（Aider 除外，走 repo map＋手動管理）。
5. 某種審批機制（Aider 除外）。
6. Headless／腳本化輸出（多數有 JSON lines）。
7. 外掛／skills 擴充（Aider 較弱）。
8. Plan 與執行分離（OpenCode Plan、Cline Plan／Act、Kilo Plan、Codex Plan mode）。

另外兩個新趨勢：**匯入競品設定**（Codex `/import` 匯入 Claude Code、Kilo 的 Cursor／Windsurf 規則遷移、Kilo `/resume-claude`），以及 **repo 內 Markdown＋frontmatter 定義 agent**（OpenCode、Kilo）。

## 7. 缺口＝我們的機會

| 缺口 | 現況證據 | 方向 |
|---|---|---|
| **預設安全＋可靠復原同時成立** | 見 §3 觀察 2、3 | 預設自動 checkpoint（不依賴使用者 git、不污染 staging）＋預設保守權限＋能降級的隔離（sandbox 失敗時不要直接退到全開） |
| **bash 副作用不可復原** | OpenCode 明確只還原檔案；其他家同樣沒處理套件安裝、資料庫、repo 外檔案 | 危險操作前的分級確認，並誠實告訴使用者哪些「還原不了」 |
| **成本可控，且不綁單一 gateway** | 只有 Kilo 有，而且綁自家 gateway | 預算上限、超限前警告、依任務選模型，做成 client 側功能而非 gateway 功能 |
| **新手入口與開源中立的矛盾** | 免 key 入口全都導向商業 gateway；Kilo 自己警告免費端點可能記錄 prompt | 首跑畫面揭露資料流向；gateway 協議開放、預設不綁定 |
| **卡住時沒人帶** | 只有 doom-loop 偵測（OpenCode、Kilo 預設 ask），沒有「換策略／白話說明」 | 偵測後主動換策略、用白話解釋卡在哪 |
| **白話層** | 輸出都是 diff 與 log；Goose 的 Recipe 仍偏 YAML | 在 diff 之上加白話摘要，Recipe 要能在 GUI 裡完整建立 |
| **穩定性與版本治理** | 5 家近期大改版／回滾／停維護 | schema 版本化、穩定通道、不要破壞性變更 |
| **評測** | 只有 OpenHands 有完整評測基礎建設（benchmarks repo）；Aider 有 edit-format leaderboard | 從第一天建立可重複評測，含「新手 10 分鐘內做出能動的東西」之類的體驗指標 |

## 8. 值得抄的設計（去重後）

1. 單一 headless server＋SDK，多前端共用（OpenCode、Cline、Kilo、Codex、OpenHands）。
2. Shadow-git checkpoint，每次 tool use 一次、涵蓋未追蹤檔、多種還原粒度（Cline）；snapshot 存在 repo 外（Kilo）。
3. `/undo` 同時還原使用者訊息，讓人能改寫重送（OpenCode）。
4. Sandbox 預設開、預設無網路、`.git` 與設定目錄唯讀（Codex）；execpolicy 規則附 `match／not_match` 範例，載入時自我測試（Codex）。
5. `.env` 預設 ask、doom-loop 偵測（OpenCode、Kilo）。
6. 依模型切換編輯工具（Cline：editor vs apply_patch）與 edit-format benchmark（Aider）。
7. Repo map：tree-sitter＋PageRank＋token 預算二分搜尋（Aider）。
8. Append-only event log＋非破壞性 condensation（OpenHands）。
9. OAuth 一鍵上手（Goose、Kilo）。
10. 相容 `AGENTS.md`／`.cursorrules`／`.claude/skills`，並提供匯入功能。
11. `codex doctor` 這類診斷指令與啟動警告。

## 9. 該避開的坑

1. 預設全 allow、CLI 預設自動核准、預設 autonomous。
2. 把權限系統當安全邊界，卻不是 sandbox；權限判斷交給模型（Cline）。
3. Undo 要求使用者有 Git（OpenCode、Codex）。
4. Sandbox 失敗時只剩「完全開放」一條路（Codex Windows）。
5. 入口和計費模式太多、預設模型頻繁更換（Cline）。
6. 大改寫整包切換後回滾（Cline 4.0.1）；fork 大型 upstream 的長期合併負擔（Kilo）。
7. 功能名稱頻繁更換（Kilo：modes→agents、memory bank→AGENTS.md→Kilo Memory）。
8. Subagent 一次核准整批放行（Cline）；subagent 不隔離檔案（Kilo）。
9. 設定面膨脹，且 UI 顯示與實際行為不一致（Codex granular approval 選了詢問卻直接拒絕）。
10. 單一維護者（Aider）。

## 10. Hermes Agent 補充

查證日期 2026-10-26。Hermes 是前面七家以外的另一種東西：**長駐的通用個人 agent**，coding 只是其中一個 toolset。它跟我們的目標使用者（想升級的 vibe coder）不是正面競爭，但它的「學習迴圈」是其他七家都沒有的，值得單獨拆解。

### 10.1 基本資料

| | Hermes Agent |
|---|---|
| Repo／授權 | NousResearch/hermes-agent，MIT |
| 技術棧 | Python 核心；Ink TUI；Electron 桌面 app |
| 規模 | repo 頁面顯示約 257k stars、59k commits、open issues／PR 各「5k+」（第二手來源的 star 數彼此矛盾，這裡只採 repo 頁面） |
| 版本 | v0.22.5（2026-10-22），每週多版，**仍是 0.x**；v0.22.1 曾因 build 錯 commit 被取代 |
| 營利 | Nous Portal 訂閱（300+ 模型＋ Tool Gateway）。融資與價格未驗證 |
| 入口 | CLI、TUI、桌面 app、ACP（VS Code／Zed／JetBrains）、約 20 個訊息平台 gateway（Telegram、Discord、Slack 等） |
| 工具 | README 寫「40+」，文件實列 71 個，但含大量平台專屬工具（kanban、feishu、desktop_ui）；一般 coding 用到的遠少於 71 |

### 10.2 學習迴圈（核心差異）

| 機制 | 做法 | 護欄 |
|---|---|---|
| 記憶 | `MEMORY.md`（約 2.2k 字元）＋ `USER.md`（約 1.4k 字元），超額必須自行整併 | session 開始時凍結進 prompt（保 prompt cache），**學到的東西下個 session 才生效**；寫入有注入／憑證外洩掃描 |
| 過往對話 | SQLite FTS5 全文檢索全部 session | 文件稱不呼叫 LLM；README 與 sessions.md 對此說法矛盾 |
| Skill | agent 用 `skill_manage` 把摸索出的流程存成 `SKILL.md`，背景 review 會建議或更新 | `write_approval: true` 時先進暫存區，用 `/skills approve` 審核；**預設是否開啟未驗證，推論為關閉** |
| Curator | 閒置時整理：14 天未用標 stale，30 天未用移入 archive | **只 archive 不刪除**；整併前有全樹 snapshot、逐筆 ledger、`curator rollback` |
| 使用者控制 | 直接編輯檔案、`/memory pending`、`hermes journey list／delete／edit` | memory 刪除沒有還原 |

**可借鏡：** 記憶分兩層（小而有上限的常駐記憶＋無上限但按需檢索的全文搜尋）；強制容量上限逼 agent 整併；暫存審核佇列；只 archive 不刪除；每筆變更有 ledger 可回滾。
**風險（報告推論，非實測）：** 自動建立的 skill 之後每次對話都可能被載入；掃描只涵蓋已知樣式；「錯但常用」的 skill 因為使用頻率高反而不會被淘汰；沒有看到對 skill 品質的自動驗證。官方「唯一內建學習迴圈」是行銷語，**沒有效果評測**。
**對新手產品的含義：** 暫存審核應預設開，並以白話介面呈現（「我學到了這個，要保留嗎？」）；新學到的東西要標示來源對話，並有一鍵停用與回滾。

### 10.3 安全與復原（Hermes 在 §3 表格中的位置）

| 項目 | 現況 |
|---|---|
| 預設核准 | **smart**：輔助 LLM 判斷風險，低風險自動核准、危險自動拒絕、不確定才問 |
| 已知事故 | open issue #141346：smart approval 自動核准了 `osascript` 指令，關掉使用者未存檔的 PowerPoint 並刪除唯一的 PDF，**資料無法復原**。社群修補 PR 被 reviewer 指出可用 base64／awk 繞過，尚未合併 |
| 硬性防線 | hardline blocklist（`rm -rf /`、fork bomb、raw disk write）即使 `--yolo` 也不繞過；`write_file`／`patch` 不能碰 `~/.ssh/`、`.env` |
| 隔離 | terminal backend 可選 local、Docker、SSH、Modal、Daytona 等。**容器 backend 視容器為邊界，略過危險命令檢查**。預設是否為 local 為推論 |
| 檔案復原 | shadow git checkpoint＋`/rollback`（可單檔、還原前自動備份、辨識使用者手改）。**但 v2 起預設關閉**（opt-in），容器 backend 不支援，預設每專案 20 份／500MB／7 天 prune，無 git 則停用 |
| 訊息 gateway | 未設 allowlist 時預設**拒絕所有人**；DM pairing 碼 1 小時過期有速率限制 |

**結論：** Hermes 把「LLM 當風險判官」做成預設，並已經造成一起無法復原的事故；checkpoint 設計得不錯但預設不開。這再次印證 §7 的缺口：**預設安全＋可靠復原**仍然沒有人做到。對我們的啟示是用確定性規則加上「GUI app 狀態、未存檔工作、刪除非自建檔案」升級人工確認，並 fail closed。

### 10.4 其他

- **規則檔：** 第一個命中即用，順序為 `.hermes.md` > `AGENTS.override.md` > `AGENTS.md` > `CLAUDE.md` > `.cursorrules`；`hermes import-agent` 可匯入 OpenClaw、Claude Code、Codex CLI、Cursor 的設定／記憶／skills。
- **壓縮：** 預設在 context 的 50% 觸發，舊 tool 結果先換佔位符，中段用結構化模板摘要。**摘要模型的 context 必須不小於主模型，否則中段被直接丟棄**（靜默失敗）。
- **Subagent：** `delegate_task` 預設最多 10 個並行，可選 git worktree 隔離，可指定較便宜的模型。
- **成本：** 沒有找到全域花費上限；有 goal 迴圈 turn budget（預設 20）、委派深度與並行上限、MCP spend limits。桌面版切到昂貴模型前會警告。
- **新手友善度：** 一行安裝＋桌面 app＋ Portal OAuth 門檻低，桌面有 Basic／Elite 版面。但表面積極大（71 工具、數十個設定頁、plugins、profiles、kanban），簡化模式是否有效收斂**未驗證**；coding 專屬品質（編輯成功率、diff review）**沒有數據**。
- **其他痛點（issue 標題層級）：** 桌面同一回覆渲染兩次（#127665）、`hermes update` 後 gateway 警告持續（#107402）、godfile 重構中（#78647，82 則留言）。

### 10.5 Hermes 對整體結論的影響

1. 「八家裡沒有任何一家同時做到強隔離、可靠復原、成本可控、預設安全」仍然成立。
2. **新增一個缺口：自動學習的狀態沒有人做好治理。** Hermes 有治理工具（暫存審核、curator、ledger），但預設寬鬆。對 vibe coder 來說，「越用越懂我的專案」很有吸引力，同時是新的風險面。這是潛在的差異化點，也可以是明確要避開的範圍，取決於你的 MVP 取捨。
3. 它不是新手 coding 產品。通用助理的表面積和 0.x 的高頻重構，跟我們想做的「新手一鍵開始、專業可深入」正好相反。

## 11. 本研究的限制（決策前需補查）

- 全部官方網站都沒讀到，只讀 repo 內文件；實際使用體驗**沒有人實測**（首跑畫面、錯誤訊息、速度、模型表現都只是文件推論）。
- 各家 benchmark 分數沒有可比較的數據，本文不評論「誰寫程式比較強」。
- Issue 內文幾乎沒讀，痛點的頻率與嚴重度無法量化。
- 閉源參考（Claude Code、Lovable、Bolt、Cursor）這輪沒有深入研究，不在本比較範圍內。
- 建議下一步：挑 2–3 家（候選：Cline、Kilo、Codex）實際安裝跑同一個新手任務，錄下首跑體驗與出錯情境，補上這份文件缺的一手觀察。
