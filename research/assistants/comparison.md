# 個人助理側：跨產品比較

查證日期：2026-10-26。資料來自 `research/assistants/` 下 8 份報告（openclaw、nanoclaw、zeroclaw、vellum、claude-cowork、chatgpt-agent、manus、zo-computer），另外參照 `research/hermes-agent.md` 與 `research/goose.md`。

## 0. 可信度說明（先讀）

| 產品 | 資料品質 |
|---|---|
| OpenClaw、NanoClaw、ZeroClaw、Vellum | 良好：直接讀到 repo 原始碼／repo 內文件，OpenClaw 還讀到 GitHub security advisory 頁面。但官方網站都連不上，且**沒有人實際安裝執行過**，預設值與 UX 全是文件推論 |
| Claude Cowork | 中等：官方 support／engineering 頁面有讀到，但經 WebFetch 摘要，非逐字原文 |
| Manus、Zo Computer | **偏低**：官方網站完全連不上，所有「官方」內容來自搜尋摘要 |
| ChatGPT agent／Work | **最低**：OpenAI 全部網站連不上，沒有打開過任何一頁原文，只能當待驗證線索 |

統一提醒：star 數、版本、價格變動快，且多半取自 WebFetch 摘要；兩份報告（Manus、Zo）的價格頁與規格互相矛盾，我沒有強行裁決。

## 1. 產品分成三種形態

| 形態 | 代表 | 使用者是誰 |
|---|---|---|
| **A. 自架的常駐 gateway**（本機或自己的主機，經聊天軟體互動） | OpenClaw、ZeroClaw、NanoClaw；Hermes Agent 也屬此類 | 會裝 Node／Docker／用終端機的 operator |
| **B. 桌面 app＋託管雲的個人助理** | Vellum、Claude Cowork、ChatGPT Work | 一般人／知識工作者 |
| **C. 託管的「雲端電腦」型自主 agent** | Manus、Zo Computer | 想要「交辦就完成」、不想管環境的人 |

Goose 介於 A 與 B 之間（桌面＋CLI 共用核心，OAuth 一鍵上手，但預設最寬權限）。

## 2. 基本資料

| | 授權／形態 | 規模與動態 | 營利 | 備註 |
|---|---|---|---|---|
| OpenClaw | MIT；TypeScript／Node | 約 39 萬 star（摘要值）；2026.10.7；約每週一版 | 無付費方案；OpenClaw Foundation（501(c)(3)）由多家企業捐助 | 名稱沿革 Warelay→Clawdbot→Moltbot→OpenClaw（2026-01-30）；2026-02 曾因 PR 湧入啟動 Stabilisation Mode |
| NanoClaw | MIT；Node＋Bun | 約 3.1 萬 star；v2026.10.0 | 未見；背後公司未驗證 | 自稱「小到能讀懂」的 OpenClaw 替代品，**非 fork** |
| ZeroClaw | MIT 或 Apache-2.0；Rust | 約 3.3 萬 star；v0.8.6（仍 0.x，有破壞性變更） | 未見；維護者「學生主導」不具名 | 獨立重寫；官方 README 不提 OpenClaw；大量同名冒牌 repo，官方自己發警告 |
| Vellum | MIT；TypeScript／Bun＋Electron | 約 1.4–1.5 千 star；約每 3 天一版；30k commits | 營利公司；託管 Vellum Platform（免費 Base；Pro $30／$100／$200） | 原本做企業 LLM 工作流平台，2026-05 左右轉向助理（二手資料） |
| Claude Cowork | 閉源（Anthropic） | 2026-04-09 GA；2026-09-16 與 chat 合併成單一「Claude」 | 含在 Pro／Max／Team／Enterprise 方案 | 變動極快，約每月一次重大調整 |
| ChatGPT Work | 閉源（OpenAI） | 2026-07-09 推出，取代 agent／Operator | 含在付費方案，用量與 Codex 共用額度 | agent 是否仍存在，官方頁面自相矛盾 |
| Manus | 閉源 | 2.0 新架構（發佈年份未驗證） | credits＋訂閱（Pro 起 $20／月；Cloud Computer $50／月） | 所有權劇烈波動：Meta 收購（2025-12）→ 中國禁止（2026-04）→ 拆解 → 2026-09 恢復獨立（單一來源） |
| Zo Computer | 閉源，**不可自架** | 營運中 | 訂閱（Basic $18／Pro $64／Ultra $200）＋ AI credits | 公司 Substrate，Lightspeed 投資；價格頁與計費頁互相矛盾 |

## 3. 威脅模型與權限（最重要）

| | 預設姿態 | 隔離 | 憑證 | 誰能對它下指令 | Prompt injection |
|---|---|---|---|---|---|
| **OpenClaw** | **sandbox 預設關**；`exec` 在宿主機、**預設無核准**（`mode=full`）；跨 session 預設互相可讀、跨 channel 預設可發訊 | 選用 sandbox；plugin／skill 以完整權限 in-process 執行 | **明文落在 `~/.openclaw/`**，靠檔案權限；transcript 與工具結果保留憑證原文 | 陌生人預設走 pairing 碼（未核准前不處理） | 官方：「純 prompt injection 不算漏洞」；用邊界標記＋建議 reader agent；承認適應性攻擊成功率可 >80% |
| **NanoClaw** | 容器內操作免核准（隔離就是邊界）；需憑證的外部動作要 approver 在 DM 核准；失敗一律 deny | **每個 session 一個容器**；只掛載明確 allowlist；host 原始碼與 `.env` 從不掛載 | **真憑證不進容器**，由 credential gateway 在網路邊界注入；沒有 gateway 就拒絕啟動 | 需成員身分；陌生人依 policy | 承認仍是風險；**egress 預設開放**、資源上限預設無，皆官方承認 |
| **ZeroClaw** | `supervised`：低風險自動、中風險問、高風險擋；逾時視為拒絕 | 每次工具呼叫套 OS sandbox（Firejail／Seatbelt）；**Windows 無 OS 級後端**；偵測不到後端時**靜默退回應用層檢查**；Firejail 不限制網路 | sandbox 移除父行程 secrets；出站回覆做憑證洩漏偵測 | Telegram 配對碼；頻道身分尚未對映到 principal 模型 | 樣式比對；官方承認新型攻擊可能通過 |
| **Vellum** | 預設 Default 風險檔：Low 自動、Medium／High 要問；排程任務預設 Relaxed | 工作區沙箱；**原生本機模式沒有 OS 隔離**，容器隔離「即將推出」 | 獨立 CES 行程，模型看不到明文；**但指令輸出會回到模型上下文** | 外部陌生人硬拒絕，不經模型 | 文件宣稱「無法 prompt-inject 繞過權限」（廠商說法）；v2 flag 把多數工具改成模型自行判斷 |
| **Claude Cowork** | **Auto 為 Pro／Max／Team 預設**（分類器審查動作），Enterprise 預設 Manual；刪除永遠要明確同意 | 雲端隔離環境；computer use **無 sandbox**（廠商自承） | 未詳述 | 帳號本人 | 訓練＋內容分類器＋動作審查；自承「機率仍非零」；2026-01 PromptArmor 外洩事件（二手），Anthropic 其後修補 |
| **ChatGPT Work** | 預設「Important actions」：讀取自動、有外部影響或難復原才問 | 雲端 container；桌面可用本機檔案 | 登入走安全流程，模型看不到 | 帳號本人 | 訓練＋監控＋watch mode＋Lockdown Mode；自承防護不能消除所有風險（**全為搜尋摘要**） |
| **Manus** | 每 task 一個雲端 VM；敏感輸入由人接手；**排程任務可 Skip confirmations** | 雲端 sandbox | 登入由使用者接手 | 帳號本人 | **找不到官方說明** |
| **Zo** | **沒有全域確認設定**，安全交給使用者自己寫 Rules 和 persona 工具範圍 | 專屬雲端容器 | 找不到 secrets 儲存文件 | 帳號本人 | 安全頁**沒有列任何防禦**，且說這類漏洞通常不給賞金 |

**這張表最重要的三個觀察：**
1. **只有 NanoClaw 同時做到「真憑證不進 agent 環境」＋「fail closed」＋「預設隔離」**，但它的網路出口預設是開的，所以被注入後仍可外洩 workspace 內的資料。ZeroClaw 和 Vellum 各做對一部分，各有靜默降級或原生模式無隔離的洞。
2. **所有產品都承認 prompt injection 沒解決**；差別在於誰把它當成需要結構性防護（NanoClaw、Vellum 的 gateway 與憑證分離），誰把它列為「不算漏洞」（OpenClaw）。
3. **Cowork 與 Vellum 都有往「模型自己判斷是否放行」移動的趨勢**（Cowork 的 Auto 預設、Vellum 的 permission v2 flag）。Hermes 的 smart approval 已經因此造成無法復原的事故（見 `comparison.md` §10.3）。這是一個行業共同壓力：核准提示太多 → 使用者麻木 → 廠商改用模型判斷 → 風險回流。

## 4. OpenClaw 的安全事件史（一手來源，這是整個調查最有價值的資料）

OpenClaw 是最大的樣本（約 39 萬 star，合理推測有大量真實使用者），GitHub advisory 頁有約 730 筆（由頁數推算，非明列）、其中 14 筆 Critical。**數量大的部分原因是維護者把邊界缺陷都公開成 GHSA，不全是被實際利用的事件。**

| 事件 | 時間 | 教訓 |
|---|---|---|
| GHSA-g8p2：Control UI 接受 URL 參數 `gatewayUrl`，自動連線並把 gateway token 送給攻擊者 → 1 次點擊 RCE（CVSS 8.8），**即使僅 loopback 也可** | 2026-01 | UI 不可被 URL 參數改寫連線目標；loopback 不等於安全 |
| GHSA-gv46：`node.invoke`→`system.run` 未剝除內部 `approved` 欄位，可繞過 exec 核准（CVSS 9.9） | 2026-02 | 核准旗標不能由 client 傳入 |
| GHSA-qrq5：plugin 安裝路徑穿越（9.3） | 2026-02 | 擴充安裝是攻擊面 |
| 2026-03 一整串 Critical：pairing／scope 一路升級到 `operator.admin` 再達 node RCE（至少 6 筆） | 2026-03 | **授權 scope 太多且互相可升級**，是最典型的結構性病灶 |
| GHSA-g5cg：heartbeat 繼承 context 經 `senderIsOwner` 繞過 sandbox（CVSS 4.0 9.5） | 2026-03 | 自動執行路徑（cron／heartbeat）不得繼承比設定者更高的權限 |
| GHSA-8rh7：啟動時載入 CWD 的 `.env`，惡意 repo 可覆寫 runtime 安全設定（9.6） | 2026-03 | 不從 CWD／workspace 載入會影響安全策略的設定 |
| GHSA-9p3r：remote FS bridge `readFile` TOCTOU → sandbox 逃逸 | 2026-03 | sandbox 本身也會有洞 |
| ClawHub 惡意 skill（"ClawHavoc"，**二手**：Koi Security 稽核 2,857 個 skill，341 個惡意） | 2026-02 | 開放 skill 市集＋完整權限＝供應鏈災難；偵測式掃描不足 |
| 暴露實例（**二手**，數字 1.7 萬到 4 萬不等、互相矛盾） | 2026 | 數字不可靠，但方向一致：很多人把 gateway 暴露在公網 |

另外兩個要注意：
- **CVE-2026-25253** 這個編號只在二手來源出現，GitHub 一手頁面顯示 "No known CVE"，兩者的對應**未驗證**。
- 部分二手文章稱「預設 bind 0.0.0.0」，與一手文件不符：host 安裝預設 loopback，**只有容器映像預設 exposed bind**。

近三個月的 Critical 集中在 **scope／pairing／approval／sandbox 的授權邏輯**，而不是單點注入。這是「有太多權限邊界要維護」的典型症狀，也是設計層的教訓，不只是補洞問題。

## 5. 復原與可稽核性

| | 檔案／動作復原 | 稽核紀錄 |
|---|---|---|
| OpenClaw | **無通用 undo**。只有局部：新 workspace 自動 `git init`、Skill Workshop 變更可 undo、MEMORY.md 重寫保留 preimage、`openclaw backup create` | **metadata-only audit ledger**（刻意不存 prompt、參數、結果）＋可匯出 trajectory＋ OTel；但 transcript 保留原文含憑證 |
| NanoClaw | 只有**升級**有快照與還原；無文件記載的回滾 | 每 session 的 DB 保存訊息歷史；**沒有獨立的核准稽核日誌**（所讀文件未提） |
| ZeroClaw | **無**（文件找不到 rollback） | JSONL 日誌，預設只滾動 200 事件；官方明說**指令執行不被稽核**；README 的「加密工具收據」只是記憶體內證明，不是稽核 |
| Vellum | 無明確使用者級 undo；ARCHITECTURE 提到 turn-boundary git commit，**用法未驗證**；備份／搬移功能有 open issue | **`tool_invocations` 稽核表**（決策、風險等級、遮蔽後輸入、結果預覽）；workspace 是純文字 |
| Claude Cowork | **未見 undo**；唯一防線是刪除需明確同意與掛載模式「讀寫不可刪」 | 管理員層：Compliance API／OTel；個人使用者的 action log 未見 |
| ChatGPT Work | 未見 undo（未驗證） | 管理員層 Compliance API |
| Manus | 無 task 級 undo／replay；網站專案只有 Make a Copy | 未找到使用者可匯出的稽核日誌 |
| Zo | 定期快照＋整台電腦可匯出 container image；停付後可用 recovery session 取回 | 未找到 |
| *Hermes（對照）* | shadow-git checkpoint＋`/rollback`，**但預設關閉** | — |

**結論：「撤銷已經發生的事」在這個領域幾乎是空白。** 看得到發生什麼（稽核）做得比能還原好得多；而 OpenClaw 的稽核還刻意不記內容。

## 6. 成本

| | 顯示 | 硬上限 |
|---|---|---|
| OpenClaw | `/usage`、Control UI Usage 頁，可接 Admin API 看實際花費 | **未找到**（單次 agent 預設 timeout 48 小時） |
| NanoClaw | 未見 | 未找到；有排程節流（每日 4 次） |
| ZeroClaw | `costs.jsonl` 帳本、每日／月預算 | `enforcement.mode` **預設 `warn`（只警告）**，可設 `block`／`route_down` |
| Vellum | 餘額、分項用量、**每日 credit 上限**、低餘額警示 | 每日上限（managed）；BYOK 自行負責 |
| Cowork／ChatGPT Work | 事後在 Usage 查看；共用額度池，5 小時視窗＋週上限 | 方案額度；Enterprise 可設群組花費上限 |
| Manus | credits 不透明；無固定單任務價 | 無；月度 credits 不滾存；退款條件嚴格 |
| Zo | 固定訂閱含運算＋ AI credits 另計 | 可關閉 AI 花費；BYOK |

**Vellum 的「每日上限＋分項用量」是目前最完整的使用者側成本控制**，ZeroClaw 的 `route_down`（超支時換便宜模型）是值得借的點子，但它預設不啟用。

## 7. 上手門檻與「替使用者做出東西」

| | 非技術使用者的首跑 | 能做出什麼 | 使用者最後擁有什麼 |
|---|---|---|---|
| OpenClaw | 一行指令；要裝 Node、取 key、建 bot token、理解 loopback、**自己打開 sandbox 和 approvals**；升級常出事（issue 可見） | 寫碼（委派給 Codex／Claude Code）、cron／webhook 自動化、`show_widget` 小儀表板、agent 自己維護 skill | 全在本機 Markdown／SQLite；但**沒有應用層的版本控管、測試、回滾** |
| NanoClaw | 約 4 分 42 秒腳本安裝；需 Docker、Claude Code、credential gateway；Windows 要 WSL2 | 容器內可寫碼、排程；**只能碰掛載進來的目錄** | 本機；每個安裝是一份 fork，升級衝突自己處理 |
| ZeroClaw | 單一 binary，`quickstart` 問 provider／模型／risk profile；不需 Docker | 直接在 workspace 內執行 shell／寫檔／git | 本機；但 supervised 下做一個 app 會產生大量核准提示，使用者傾向切到 YOLO |
| Vellum | 下載 app→註冊→**預設連雲端助理，約 30 秒**；沒有 wizard，只有聊天框；macOS 權限用到才要 | **App Builder** skill（小工具／儀表板）、排程、workflow（QuickJS-WASM 沙箱） | workspace 是人可讀文字檔；產出品質**未實測** |
| Cowork | 開 Claude→描述任務→授權資料夾 | 文件、Excel、簡報、live artifacts；開發另走 Claude Code | 檔案在你的資料夾；**artifacts 跑在 Anthropic 基礎設施** |
| ChatGPT Work | 統一 app；逐一授權資料夾／網站／app | 文件、簡報、**ChatGPT Sites**（託管網站／輕量 app，beta） | 託管在 OpenAI |
| Manus | 自然語言交辦；登入時 Take Over | **WebDev：一鍵發佈、自訂網域、內建資料庫**；Cloud Computer 跑 bot／腳本 | **程式碼可匯出（含 GitHub 雙向同步），但託管、資料庫、webhook 綁在 Manus**；官方自己說匯出「不是完整備份」；2026-08 有帳號資料被刪除、網站下線的事件 |
| Zo | 簡訊／聊天指揮；「不像一般 SaaS 即開即用」（次要評測） | Zo Sites（React app、API、資料庫、長駐程序）、排程 agent | **檔案與 container image 可取回**；但 Zo 本身不能自己跑，服務與網域路由依賴 Zo |

## 8. 共通點（入場門票）

這一側幾乎所有產品都有：
1. **多通道**（聊天軟體／簡訊／email）加上桌面或 web 控制台。
2. **持久記憶**：多數用人可讀的 Markdown（OpenClaw、NanoClaw、Vellum、Hermes）。
3. **排程與事件觸發**的自動化。
4. **Skills／plugins／MCP** 擴充。
5. **分級核准**：讀取自動、有外部影響才問；刪除與付款永遠要人。
6. **登入／付款由人接管**（Manus、ChatGPT agent、Cowork 的 computer use 皆有）。
7. **誠實承認 prompt injection 未解**（唯一例外是把它「不算漏洞」的 OpenClaw，但它也承認風險）。
8. **匯入競品設定**（Hermes、Codex、Kilo 前面已見）。

## 9. 缺口

| 缺口 | 證據 | 機會 |
|---|---|---|
| **預設安全＋可回復＋非技術者能用**三者同時成立 | NanoClaw 安全但重；Vellum 友善但原生模式無隔離；OpenClaw 最強大但預設最危險；沒有人把可回復做進預設 | 這是最清楚的空位 |
| **通用 undo／可回滾的副作用** | §5 | 檔案 snapshot（不依賴使用者 git）＋對「還原不了」的動作（寄信、轉帳、外送）事前明示並逐次核准 |
| **網路出口控制** | NanoClaw、ZeroClaw、Vellum 全部預設不鎖；Cowork 自承 egress 設定不涵蓋 web fetch／MCP | 預設全關、由使用者逐一開啟，並把「開放哪些網域」做成白話介面 |
| **所有權與可攜性** | Manus 的託管資料不可攜且有刪除事件；Zo 不能自架；Cowork／ChatGPT 的 artifacts 在廠商基礎設施 | 產出物（程式碼、資料、排程、規則）預設存在使用者可控的檔案系統，用開放格式 |
| **成本可預測** | ZeroClaw 預設只警告；OpenClaw 無硬上限；Manus credits 不透明 | 預設硬上限＋任務前預估＋ `route_down` |
| **核准疲勞的解法** | Vellum 官方文件自己警告使用者會麻木連點；Cowork 預設 Auto；Hermes smart approval 出過事故 | 用**確定性規則＋風險分級＋「Allow once」為預設、建立永久規則多一步**，而不是改用模型判斷 |
| **自動學習的治理** | OpenClaw 有 Skill Workshop undo、Hermes 有 curator，但預設寬鬆 | 新學到的內容預設進暫存審核區 |
| **可稽核但不外洩** | OpenClaw 的 ledger 只存 metadata（好），但 transcript 存憑證原文（壞） | 稽核記錄與敏感內容分離 |

## 10. 從事件史推導的最低安全門檻

這是我（非廠商）從 OpenClaw 的一手 advisory 加上其他產品的自承限制推導的，**屬於推論，需要之後用威脅模型驗證**：

1. 預設只綁 loopback，且 UI 不可被 URL 參數改寫連線目標。
2. **不信任內容與外送能力不得同時存在於同一個 agent 上下文**（reader／actor 分離）。
3. 外部副作用（寄信、刪檔、轉帳、發訊息）預設逐次核准，核准內容綁定實際參數。
4. 任何自動執行的路徑（cron、heartbeat、hook）**不得繼承比設定者更高的權限**。
5. 不從 CWD 或 workspace 載入會影響安全策略的設定。
6. **憑證不與 agent 同一個可讀範圍**（OS keychain 或獨立 broker）；指令輸出要過濾。
7. 第三方擴充必須隔離執行，不預設安裝；偵測式掃描不夠。
8. 授權 scope 盡量少，避免互相可升級的多層角色。
9. 檔案變更可還原，並有硬性花費上限。
10. sandbox 偵測失敗時**大聲失敗**，不要靜默降級。
11. 內建自檢指令（OpenClaw 的 `security audit` 值得抄），並在首次設定時用白話展示風險摘要。

## 11. 對定位決策的含義

上一輪我建議的方向是「會替你做出並維運個人軟體的助理」。研究後我的看法有所更新：

**支持這個方向的證據**
- 需求真實且巨大（OpenClaw、Hermes 的規模）。
- 「安全＋可回復＋可攜」這個組合確實沒有人做好，而且每一項都有具體、可引用的失敗案例。
- 「做出東西」這條軸上，閉源產品（Manus、Zo、ChatGPT Sites）的共同弱點是所有權，這正是開源自架的天然優勢。

**要誠實面對的風險**
1. **最近的對手已經很近：Vellum。** MIT、桌面 app、四檔風險等級、確定性的 gateway 分類、獨立憑證服務、每日 credit 上限、tool 稽核表、App Builder，且面向「一般人」。它的缺口是：原生模式無 OS 隔離、容器隔離「即將推出」、文件前後矛盾、產品重心在付費託管版、undo 不明確。所以**不是空地，差異化要靠它沒做好的幾點**，而不是靠「安全優先」這個口號本身。
2. **這個賽道的安全工程量遠大於 coding harness。** OpenClaw 用 730 筆 advisory 示範了常駐助理的攻擊面有多大。即使只做最小版本，也必須先把 §10 的門檻當成不可妥協的需求，而不是之後補。
3. **非技術使用者與安全預設是互相拉扯的。** NanoClaw 與 ZeroClaw 的教訓：檔案與憑證隔離容易，**網路出口與核准提示量**才決定使用者會不會關掉安全設定。這不是工程問題，是產品設計問題。
4. **本調查沒人實測。** 首跑體驗、核准提示的實際頻率、App Builder 的產出品質，全是文件推論。

**所以我現在的建議，從兩條路收斂成兩個可比較的選項：**

| | 選項一：安全優先的 coding harness（原本的方向） | 選項二：安全優先、可攜的「個人軟體助理」 |
|---|---|---|
| 目標使用者 | 想升級的 vibe coder | 想擁有自己的小工具和自動化的非工程師 |
| 最近的對手 | Cline、Kilo、OpenCode | Vellum、Zo、Manus、OpenClaw |
| 安全工程量 | 中（專案範圍，單次工作階段） | **高**（常駐、持有憑證、讀外部內容、多通道） |
| 差異化 | 預設 checkpoint、保守權限、成本上限、卡住引導 | 所有權與可攜、可回復、egress 預設鎖、確定性核准 |
| 風險 | 賽道已有 8 家成熟開源 | 賽道更擠且安全門檻更高；Vellum 先到 |
| 可以先做小版本驗證？ | 容易 | 較難；但可以先做「單通道、無常駐」的最小版 |

**一個折衷的做法**：先以選項一的範圍起步（本機、單次 session、沒有訊息 gateway 和常駐排程），但**架構上就採用選項二需要的安全模型**（確定性權限、憑證分離、egress 預設鎖、可還原的 snapshot、所有產出物存在使用者的 repo）。這樣可以先在較小的攻擊面上驗證核心，再決定要不要往常駐助理延伸。

## 12. 本研究的限制

- 官方網站幾乎全部連不上；Manus、Zo、ChatGPT 的報告建立在搜尋摘要上，不是原文。
- **沒有實測任何產品**；本比較中的 UX 判斷都是文件推論。
- Issue 與 advisory 多數只讀到標題或摘要，頻率與嚴重度無法量化。
- 暴露實例數量、惡意 skill 數量等都是二手數字且互相矛盾，請勿引用。
- 閉源產品變動極快（Cowork 約每月一次重大調整，OpenAI 一年更名三次），本文很快會過時。
- 建議下一步：挑 2–3 個（候選：Vellum、NanoClaw、Cline）實際安裝，用同一個新手任務錄下首跑體驗與核准提示頻率，補上這份文件缺的一手觀察。
