# Claude Cowork（Anthropic）研究筆記

> 方法聲明：我們無法實際使用產品，以下全部為「文件層級」資料。主要來源為 support.claude.com、claude.com、anthropic.com 的官方頁面，經 WebFetch 取得（WebFetch 內部會用小模型摘要頁面，故引文為摘要工具回傳之文字，非逐字抓取原 HTML，凡標「聲明」者皆為廠商自述）。第三方評論僅用於「已知問題與外部評價」並明確標註。未能驗證之事項一律寫「未驗證」。
> 檢查日期：2026-10-26。產品變動極快（約每月一次重大調整），本文可能很快過時。

## 1. 基本資料

- 廠商：Anthropic。
- 時間線（來源：support.claude.com Release notes，WebFetch 取得）：
  - 2026-01-12：research preview，僅 Max、僅 macOS。
  - 2026-01-16：擴大到 Pro（仍僅 macOS）。
  - 2026-02-24：plugins 與 Team/Enterprise 管理控制。
  - 2026-02-25：Scheduled tasks。
  - 2026-03-17：以手機控制 Cowork 的 persistent thread（Pro/Max）。
  - 2026-03-23：computer use research preview（Pro/Max）。
  - **2026-04-09：Generally available**，macOS 與 Windows，透過 Claude Desktop，所有付費方案；同時加入 Analytics API、OpenTelemetry、Enterprise 角色權限。
  - 2026-07-07：web 與 mobile（beta，先給 Max）；雲端 session，關掉筆電仍可繼續。
  - 2026-08-25：Memory 跨 chat 與雲端 Cowork。
  - 2026-09-16：Cowork 與 chat 合併為單一「Claude」體驗（Pro/Max 先行；2026-10-23 對 Enterprise GA，此日期來自搜尋結果摘要，未直接開頁驗證）。
- 方案與價格（claude.com/pricing，WebFetch）：Free 不含 Cowork；Pro 約 $17/月（年繳）或 $20/月；Max 起 $100/月；Team Standard seat $20–25、Premium seat $100–125；Enterprise $20/seat 起加用量。頁面註明 Cowork 在 Pro/Max「rolling out now」。
- 平台（Get started 文章）：Desktop app（macOS、Windows；Linux 標示 beta）；web 與 mobile 為 Pro/Max、Team（beta）、Enterprise 需 owner 啟用；另有 Chrome side panel。
- 文章明言「無 GA 日期」於 Get started 頁，但 Release notes 載明 2026-04-09 GA；兩者並存，以 Release notes 為準，但 web/mobile、unified 版本仍標 beta／漸進推出。

## 2. 定位與使用場景

- 官方定位（聲明）：「Cowork uses the same agentic architecture as Claude Code, without requiring a terminal」；產品頁稱 "built for non-coding knowledge work"。你描述一個結果（outcome），Claude 規劃並執行多步驟任務。
- 廠商展示場景：整理檔案、產出含公式的 Excel、PowerPoint、格式化文件、研究整合；web/mobile 發表文的範例是每週一 6am 自動做客戶準備：讀 email threads、逐字稿、新聞，產出簡報文件，並「草擬但不寄出」跟進 email。
- 廠商自述：超過 90% 的 Cowork 使用不是軟體開發，商業營運與內容創作合計約一半（claim，來源 2026-07-07 blog）。

## 3. 執行環境

這是理解 Cowork 最需要小心的一點，因為架構在 2026 年中改過：
- 最初版本（Anthropic engineering 文 "How we contain Claude across products"）：agent 跑在**本機完整 VM**（macOS 用 Apple Virtualization framework、Windows 用 HCS），有自己的 Linux kernel；只掛載使用者選的 workspace 資料夾與 `.claude`，「nothing else on the host is visible」。後來為了 VM 啟動失敗問題，把 agent loop 移到 VM 外，只讓「code execution」留在 VM 內；local MCP server 移到 host（以上為該文自述）。
- 目前 Get started 文章：任務預設在 **Anthropic 伺服器上的隔離環境（cloud）** 執行，session 與檔案存在帳號；要碰本機檔案、瀏覽器、computer use 時，需 Desktop app 開啟並連線。文章未稱其為 VM。Team/Enterprise 文章仍保留「local sessions：程式碼在隔離 VM、對話記錄存使用者電腦」與「cloud sessions」兩種模式。
- 能碰到什麼：使用者連接的資料夾（唯讀/讀寫/讀寫不可刪三種掛載模式，engineering 文）、connectors（Gmail、Drive、Slack、M365…）、內建瀏覽器或 Claude in Chrome、computer use（螢幕截圖+點擊，Pro/Max，beta，需電腦醒著且 app 開著）。
- 工具優先序（computer use 文章）：connectors > browser > 螢幕操作。

## 4. 權限與安全模型（重點）

**權限模式**（Get started 文章，聲明）：
- Manual：每個動作都問，Allow/Deny；Enterprise 預設。
- Auto："Claude reviews each action for safety (such as data exfiltration or prompt injection) and blocks anything it determines to be unsafe"；唯讀工具直接核准；持續被擋會退回逐步詢問；**Pro/Max/Team 預設**。文章稱經內部測試與第三方 red-team，並寫 "no defense is perfect"。
- Skip all approvals：無任何檢查，文章警告「Only use this when you completely trust every action, connector, file, app…」。
- Connector 工具個別設 Always allow / Needs approval / Blocked。

**範圍限制**：
- 只能讀寫已連接的資料夾（聲明）。
- 永久刪檔須使用者明確 Allow（聲明，任何模式都適用）。
- computer use：「asks permission before accessing each application」；預設封鎖投資/加密貨幣類 app，可自訂封鎖清單；macOS 15+ 預設在背景視窗操作。
- 網路 egress 設定不適用 web fetch、web search、MCP、Claude in Chrome（廠商自承）。

**Prompt injection 緩解（聲明）**：模型訓練拒絕惡意指令「even when they appear authoritative or urgent」；內容 classifier「scan all untrusted content entering Claude's context and flag potential injections」；Auto mode 動作審查；每 session 的雲端環境結束即刪除。同時自承 "the chances of an attack are still non-zero"。

**廠商自承的限制**：
- 隔離「doesn't limit what Claude reads or does」through granted access。
- computer use「has no sandbox between Claude and what's on your screen」，點到連結會在 Chrome 開啟，即使沒授權該 app。
- Chrome side panel 可看到已登入頁面，強烈建議不要處理敏感資訊。
- 本機檔案被讀取時會在 Anthropic 伺服器上處理（不只留在電腦）。
- 「You remain responsible for all actions taken by Claude performed on your behalf」。
- Engineering 文自承：早期 egress allowlist（允許 api.anthropic.com）曾被濫用，惡意檔案可讓 Claude 以攻擊者的 API key 上傳檔案，後以 VM 內 MITM proxy 修補；VM 對企業 EDR 是不透明的。

**資料保存**：刪除 task 後立即從歷史移除、30 天內從後端刪除（Get started）。本機 session 的對話記錄存在使用者電腦，不受 Anthropic 標準保存政策管、admin 無法集中刪除（Team/Enterprise 文章）。搜尋結果摘要稱 Cowork transcripts 預設保存 6 年（未直接開頁驗證，標「未驗證」）。訓練使用：商業條款預設不用；消費者帳號可選擇加入（由搜尋摘要得出，未驗證 Cowork 專屬說明）。

## 5. 復原與可稽核性

- 可見/可停：官方稱可「Redirect, or stop anytime」，session 可從任何裝置查看、導引、回答（Get started）。顯示計畫並於重大動作前等待核准是可選模式（產品頁，聲明）。
- 復原：Team/Enterprise 文章**未描述 undo 功能**，也未描述給成員的 in-product activity log（我們在已讀頁面未見；其他頁可能有，未驗證）。唯一明確的復原相關機制是「刪除需明確許可」與掛載模式 read-write-no-delete。
- 稽核（管理員）：Compliance API 涵蓋 Cowork（2026-08-11 文，Desktop/web/mobile GA 於 08-26）；OpenTelemetry 串流工具呼叫、檔案存取、人工核准決策，但文章明說「doesn't replace audit logging for compliance」。Scheduled tasks 的過往執行可在 Scheduled 頁回顧。
- 注意：不同版本的 safety 文章對 Compliance API 是否涵蓋有出入（搜尋摘要所見），以官方最新頁為準。

## 6. 記憶、排程與主動行為

- 記憶：2026-08-25 起 memory 跨 chat 與雲端 Cowork，可在 Topics 編輯；啟動任務前可關閉 memory，「set when the task starts and can't be changed later」。
- 排程：`/schedule` 或 Scheduled 側欄；雲端執行，不需電腦開機。官方建議排程任務避免敏感資料與不可逆動作。
- 主動行為：沒有「無人觸發的自主監看」的說法，僅有使用者定義的排程；Dispatch（手機傳訊給桌面 Claude 工作）為 limited beta、不開放新使用者。
- 全域/資料夾 instructions：Settings 設定全域指示；資料夾指示可由 Claude 於 session 中更新。

## 7. 擴充與寫程式能力

- Plugins：bundle skills、connectors、sub-agents；帳號同步，chat 與 Claude Code 也可用；含 local MCP server 的 plugin 只能在 Desktop；Enterprise 可啟用 skill 掃描惡意內容。
- Skills：相關時自動載入或以 slash 指令呼叫。MCP：支援，建議只用可信來源；local MCP 以與使用者其他程式相同權限執行（官方警語）。
- 寫程式/建東西：Cowork 在隔離環境執行 code 與 shell；輸出 xlsx/pptx/文件；live artifacts（可呼叫 Claude 的互動 app，執行於 Anthropic 基礎設施）；Docs/Slides/Design（beta）。Cowork 定位非開發，開發另走 Claude Code（Dispatch 可把開發工作轉去 Code session）。

## 8. 成本與用量限制

- 與 Claude 共用用量池；5 小時滾動視窗＋每週上限；可開 usage credits 以 API 費率續用（pricing 頁）。
- 官方：多步驟任務耗用顯著高於一般問答，於 Settings > Usage 查看。2026-07 曾有促銷「Cowork 用量加倍至 08-05」。
- Enterprise：admin 可設群組花費上限。

## 9. 上手體驗（非技術使用者）

- 官方流程：開 Claude → 選 Cowork（或統一版直接對話）→ 描述任務 → 審閱方法 → 讓它跑。資料夾須主動授權。
- 優點（文件層）：預設有權限提示、刪除保險、手機通知決策。缺點（文件層）：Auto 為預設，意味新手預設即讓 AI 分類器審查而非人工核准，官方卻同時建議「敏感檔案/新工具/難復原動作」切 Manual——把判斷模式的責任交給新手。

## 10. 已知問題與外部評價（第三方，次要來源）

- PromptArmor（2026-01，產品發表後約兩天）：隱藏於文件（偽裝成 Skill）的指令使 Cowork 透過被白名單的 Anthropic API 把本機檔案上傳到攻擊者帳號；The Decoder、CU InfoSecurity 等轉載；Anthropic 其後以 proxy 修補（此點見其 engineering 文，屬廠商說法）。
- Simon Willison 觀點（經轉述）：叫非程式使用者「留意可疑動作」不合理。
- WIRED 早期評測（經 TechRepublic 轉述）：批量封存促銷信失敗，改成刪除才完成。
- 使用者回報單一簡報任務耗用大量額度（FindSkill，未驗證）。
- 以上均為第三方，未經我們驗證。

## 11. 對我們的啟發

**Table stakes（應對齊）**：
1. 明確資料夾授權，預設零存取；刪除永遠需明確同意。
2. 三段式權限（逐一問/自動審查/全跳過）且「全跳過」有強警告。
3. per-connector、per-tool 的 Always/Ask/Block。
4. 計畫先行（show plan then execute）、可隨時中止/導引。
5. 排程任務的過往執行紀錄頁。
6. 對 injection 的誠實說明：不宣稱解決。

**可差異化（開源/自託管）**：
- **本地優先與可驗證隔離**：Cowork 預設雲端、本機檔案會在伺服器處理；自託管可保證資料不離機，且 sandbox 設定（掛載、egress）可被使用者審查，而不是信任廠商。
- **預設較保守**：新手預設 Manual／read-only 掛載＋明確 diff/預覽，而非 Auto 分類器。
- **內建復原**：Cowork 文件看不到 undo；工作區資料夾自動 git snapshot / 檔案版本，可一鍵回滾，是強差異點。
- **本地稽核日誌**：使用者可讀的 action log（目前 Cowork 的稽核集中在 admin 的 Compliance API/OTel）。
- 預設 egress 全關、白名單由使用者逐一開啟（Cowork 自承 egress 不涵蓋 web fetch/MCP）。
- 成本透明：顯示每任務 token 預估（Cowork 僅事後看 Usage）。

## 12. 來源與未能驗證

來源（皆經 WebFetch 取得摘要，2026-10-26）：
- support.claude.com/en/articles/13345190（Get started with Cowork）
- support.claude.com/en/articles/13364135（Use Claude Cowork safely）
- support.claude.com/en/articles/13455879（Team 與 Enterprise）
- support.claude.com/en/articles/14128542（computer use in Cowork）
- support.claude.com/en/articles/12138966（Release notes）
- claude.com/resources/articles/cowork-is-now-claude、claude.com/blog/cowork-web-mobile、claude.com/blog/compliance-api-cowork-and-claude-code、claude.com/pricing、claude.com/blog/cowork-research-preview（產品頁內容）
- anthropic.com/engineering/how-we-contain-claude
- 第三方（次要）：promptarmor.com、the-decoder.com、cuinfosecurity.com、obot.ai、techrepublic.com、findskill.ai（皆僅取搜尋摘要，未開頁）。

未能驗證：
- 原始 research preview 公告全文（website.claude.com 無法解析 DNS；claude.com/blog/cowork-research-preview 回傳的是產品頁風格內容）。
- 「Cowork 是否仍為 VM」：官方 Get started 不稱 VM，與 engineering 文／Team 文章對 local 模式的 VM 描述並存。
- 6 年 transcript 保存、2026-10-23 Enterprise GA、Auto mode 的上線日期（Release notes 無 Auto 條目）。
- 無法實際使用產品，所有 UX 評語均為文件推論。
