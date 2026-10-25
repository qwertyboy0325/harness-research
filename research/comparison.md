# 開源 Harness 跨工具比較

查證日期：2026-10-25。資料全部來自 `research/` 下 7 份個別報告（opencode、cline、kilo-code、aider、codex-cli、goose、openhands）。
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
1. 七家裡只有 Cline 與 Kilo 做了「不依賴使用者 git、不污染使用者 git」的 checkpoint；其中 Cline 較完整，Kilo 有 7 天 gc 的限制。
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

## 10. 本研究的限制（決策前需補查）

- 全部官方網站都沒讀到，只讀 repo 內文件；實際使用體驗**沒有人實測**（首跑畫面、錯誤訊息、速度、模型表現都只是文件推論）。
- 各家 benchmark 分數沒有可比較的數據，本文不評論「誰寫程式比較強」。
- Issue 內文幾乎沒讀，痛點的頻率與嚴重度無法量化。
- 閉源參考（Claude Code、Lovable、Bolt、Cursor）這輪沒有深入研究，不在本比較範圍內。
- 建議下一步：挑 2–3 家（候選：Cline、Kilo、Codex）實際安裝跑同一個新手任務，錄下首跑體驗與出錯情境，補上這份文件缺的一手觀察。
