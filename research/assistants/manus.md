# Manus 研究筆記（文件層級，非實測）

> **方法與限制（必讀）**：本檔案作者無法使用產品。官方站點（manus.im、help.manus.im、trust.manus.im）以 WebFetch 直接抓取時全部失敗（`getaddrinfo ENOTFOUND`；以 curl 經 proxy 也回 `CONNECT 403`）。因此下列「官方」內容，均來自 WebSearch 工具回傳的**搜尋摘要（對官方頁面的二手轉述）**，並附上官方 URL；我沒有親眼讀過原始頁面全文。每項標示：【官方頁面（經搜尋摘要）】、【媒體報導】、【第三方／次要】。搜尋摘要自己也多次提醒「頁面日期不一致、可能已過期」，請當作線索，採用前須回官方頁面核對。
>
> 查核日期：2026-10-26。

## 1. 基本資料

- **公司／母公司**：Butterfly Effect（Manus 母公司）；創辦人／CEO 肖弘（Xiao Hong, "Red"）。【媒體報導，多來源】
- **所有權與狀態（高度動態，務必重新確認）**：
  - 2025-12-29 Manus 宣布被 Meta 收購，報導價格逾 20 億美元（WSJ 轉述；未官方揭露財務條款）。【媒體報導】
  - 2026-04-27 中國 NDRC 下設的外商投資安全審查工作機制**禁止**該交易並要求拆解（unwind）；1 月 MOFCOM 已表示將調查。【媒體報導：Fortune、CNBC、Foreign Policy 等的標題與摘要】
  - 2026-06 起報導 Meta 依北京要求拆解（TechCrunch 2026-06-13）；Meta 於 8 月 31 日後不再控制 Manus；有分析稱 Manus 已於 **9 月 1 日正式恢復獨立營運**。【媒體／第三方分析，單一來源，未驗證】
  - 目前股權結構**未公開完整說明**：有報導稱母公司於 10 月宣布募資逾 5 億美元（Boyu、IDG 領投，Tencent 等參與），也有報導稱 Tencent 洽談成為最大股東；這些為第三方報導，**未驗證**。
  - 結論：Manus **仍在營運**，但在過去 10 個月內經歷「被收購→被禁止→拆解→獨立」，所有權是最大的不確定性。
- **可用性**：Web、行動、Telegram/WhatsApp Business agents、Desktop（"My Computer"／Manus Studio）、API（open.manus.im）。【官方頁面（經摘要）】
- **定價模型**：訂閱 + credits 計量。【官方 help center（經摘要）】
  - Free：每日 300 credits（每月上限 1,500；僅能用於 Manus 2.0 Lite），註冊一次性 1,000 credits，1 個並行任務、2 個排程自動化。
  - Pro：起價 US$20/月（年付 17% 折扣），月 4,000 credits 起；另有 US$40/月 8,000 credits 的檔位。舊 Basic/Plus 已改名為 Pro。
  - Team：起價每座位 US$20/月。（某第三方文章寫 US$200，與官方不符，以官方為準）
  - Cloud Computer（見 §3）另計：Advanced 方案 US$50/月（2 vCPU、8GB RAM、120GB）。

## 2. 定位與使用場景

- 官方定位：通用型自主 agent（"Hands On AI"），涵蓋研究（Wide Research）、簡報／文件、網站與 web app、遊戲、影片、手機 app 開發、瀏覽器操作等。【官方】
- 範例流程（官方文件列出）：每小時執行的爬蟲、24/7 Discord/Telegram bot、排程報表、發佈網站並綁自訂網域。
- **Manus 2.0**（官方部落格 "Introducing Manus 2.0"，摘要稱發佈日為 9 月 28 日，年份摘要未明示，**未驗證**）：官方稱為「新架構」而非小版本；包含新 agent harness「Cascade」（官方自測：token 少 23.2%、時間少 28.2%、成本少 32%——**僅為廠商在單一測試設定下的宣稱**）、Manus Studio（取代 Desktop app）、Cloud Computer、事件觸發的 Automations、Computer Use 遠端控制、另一款個人 agent app「Cue」。官方自承「仍有不少粗糙之處」。Free 僅能用 2.0 Lite，Pro 才有 2.0 / 2.0 Max / Lite。

## 3. 執行環境

- **預設 Sandbox（每個 task 一台）**：隔離的雲端 Linux VM，可並行；閒置休眠、保留檔案；閒置過久被回收（Free 7 天、Pro 21 天），回收後重開 task 會建新 sandbox，僅自動還原 artifacts、上傳附件與 Slides/WebDev 等重要檔案，**中間程式碼與暫存檔不還原**。官方文件對「sandbox 是否持久」的說法在不同頁面有出入（blog 稱有 persistent file system，Cloud Computer 頁面稱標準 sandbox 為暫時性）。【官方，內部矛盾】
- **Cloud Computer（持久）**：專屬、持久、Ubuntu Server 24.04 LTS 的 VM，有專屬公網 IP；檔案、已安裝工具、背景程序跨 session 保留，所有 task 共用同一顆硬碟；可跑 bot、排程腳本、託管站台。Manus 會在任務需要持久性時自動判斷並部署（需有效方案）。**訂閱失效後工作檔、工具、資料庫會被刪除**（一頁寫取消後 30 天未重新啟用即刪；官方建議取消前備份）。【官方（經摘要）】
- **Browser**：Cloud Browser（雲端）、Browser Operator（使用者本機瀏覽器擴充）；**My Computer／Desktop**：在使用者自己的電腦上、已授權的資料夾內執行指令。
- **可觸及範圍**：網路、connectors（含 GitHub 等 MCP connectors）、使用者授權的本機資料夾／瀏覽器。
- **資料位置**：官方稱資料存放於美國與新加坡。【官方（經摘要）】

## 4. 權限與安全模型

（以下均為**廠商宣稱**，我無法獨立驗證）
- 隔離：每 task 一個 sandbox，官方稱採 Zero Trust，sandbox 內操作無法觸及帳號資料。
- 認證：官方稱 SOC 2 Type II、ISO 27001、ISO 27701；與各 LLM 供應商簽有「不用於訓練」的協議；Trust Center 在 trust.manus.im。
- **敏感操作由人接手**：Cloud Browser 遇到密碼／付款輸入、SMS、CAPTCHA、MFA 時要求使用者 "Take Over"，官方稱接手期間不收集輸入；下單、寄信等「重大動作」前官方稱會徵求明確同意。登入狀態是否儲存需使用者確認，可在 Settings > Cloud Browser 關閉或逐站管理。Browser Operator 官方稱每個 session 需授權、不存取密碼（驗證留在本機瀏覽器）。My Computer：資料夾授權後只對「敏感指令」請求確認，並非每個指令。
- **例外：Scheduled Tasks 2.0 提供 "Skip confirmations"**，讓受信任流程在寄送／發佈前不再詢問——這是便利與安全的取捨，值得注意。
- **Prompt injection**：搜尋未找到官方專門說明（「未驗證／未找到」）。
- **資料保留**：刪除帳號時刪除內容，但可能為法遵、會計、防詐、請求處理保留部分個資；Team 方案由 owner/admin 設定保留。
- **廠商自承的限制／事件**：「A Note to Our Users」與 Service Change 系列 help 文章顯示：因特定司法管轄區的法規要求，特定帳號在 2025-12-29 之後產生的資料（含 WebDev 專案與網站）被排定於 2026-08-23 至 08-25（SGT）刪除，期間託管網站下線、webhook 停止且失敗請求不重送；之後開放還原入口。官方並表示**單靠 GitHub／原始碼匯出不是完整備份**，且無官方方式把整個託管 WebDev 服務搬到別處。這對「所有權」的評估非常關鍵（見 §7、§11）。受影響範圍（Type A/B/C）的細節以官方信件為準，**我無法確認哪類使用者被影響**。

## 5. 復原與可稽核性

- 觀看：task 執行過程可在介面中觀看（官方描述有 Take Over 即時接手瀏覽器）；細節**未驗證**。
- 停止：API 有 `task.stop`；task.detail 文件提示「主 agent 停止後背景工作可能仍在跑」，需檢查 background-jobs 旗標。【官方 API 文件（經摘要）】
- 重播（replay）：搜尋**未找到**此功能。
- 還原／回滾：網站專案有 Make a Copy（複製最新程式碼到新 session，不含 GitHub 連線、聊天歷史、資料庫列）；任務層級的「undo」未找到文件。
- 稽核：沒有找到使用者可匯出的完整動作稽核日誌文件（未驗證）。

## 6. 記憶、排程與主動行為

- **排程**：Scheduled Tasks 2.0——每次執行可選「延續同一 task」或「另開新 task」，可連接 connectors、Projects 的共享檔案／skills／指示；web app 也可有自己的 scheduled actions。Free 方案限 2 個排程。
- **事件觸發 Automations**（2.0）：由已連接服務的事件觸發（官方宣稱）。
- **記憶**：搜尋未找到獨立的「memory」功能頁；文件中最接近的是 Projects（共享檔案與指示）、web app 內建資料庫（長期儲存）、Cloud Computer 的共享檔案系統。個人化記憶機制**未驗證**。
- **主動行為**：Cloud Computer 可 24/7 跑 bot；Telegram/WhatsApp 中的 Manus agent 可從聊天室互動。是否會「主動提出建議」未找到文件。

## 7. 能否替使用者做出東西（關鍵軸）

- **Web app / 網站**：Manus 內建 website builder（WebDev）；預覽後一鍵 Publish，Manus 負責託管與 DNS；可連自己的網域（給 DNS 記錄，自動簽 SSL）或在 builder 內購買網域；內建資料庫、cloud infrastructure、web app 內排程動作。【官方文件（經摘要）】
- **使用者最終擁有什麼**：
  - 可隨時下載前後端完整程式碼（Code Control）；可匯出到**新建的私有 GitHub repo** 並雙向同步（僅 task owner 可匯出；改 repo 名稱／擁有者會斷線）。
  - **但**：託管、資料庫列、webhook、Forge API 等仍綁在 Manus 平台；官方在 Service Change 文件中明說 GitHub／原始碼匯出**不是完整備份**，且沒有官方遷移託管服務的方法。Free 使用者可申請「程式碼包自行部署」，完整網域支援限付費。
  - 腳本／bot：跑在 Cloud Computer（US$50/月起）；訂閱停止即刪除。要自己帶走需自行備份。
- 結論：**程式碼可攜、運行環境與資料不可攜（hosted-first）**；且已有真實的「平台一次大規模刪除／下線事件」先例。

## 8. 成本結構

- Credits 依 LLM tokens、VM 使用、第三方 API 消耗；僅在 task 主動處理時耗用；**無固定單任務價格**（官方只給三個範例：200／360／900 credits，據第三方部落格）。消耗順序：event > daily > monthly > add-on > free。
- 月度 credits 不滾存；add-on 的有效期官方頁面互相矛盾（舊頁稱永不過期，較新的 help 稱需有效付費訂閱，降級為 Free 後不可用）。
- 退款：Terms 稱 credits 購買一律不可退，僅「例外情況」酌情部分退；因 Manus 方技術問題失敗的任務會全額退還已耗 credits。
- **意外成本抱怨（次要來源）**：Trustpilot（約 151 則，摘要稱多數為負評）出現「credits 消耗過快但任務不穩」、「方案切換後 31,132 個未用 credits 在週期開始時作廢」、「購買／降級時未清楚揭露」、「退款被罐頭回覆拒絕」；eesel.ai 等部落格稱其 credit 制「有爭議」。均為軼事性質，**未驗證**。
- Cloud Computer 是否額外消耗 credits：搜尋**未能確認**（官方產品頁稱為月訂閱，儲存另計）。

## 9. 上手體驗（非技術使用者）

- 官方訴求是「用自然語言交辦」；Chat／Agent 模式、模板（playbook）、Manus Academy。Take Over 機制讓使用者在登入／驗證時介入。
- 本人無法實測；官方自承 2.0 有粗糙處。是否適合新手取決於 credits 可預期性——見 §8。第三方使用者體驗未取得可靠樣本。

## 10. 已知問題與外部評價（次要，標示）

- 所有權不確定性與地緣政治風險（媒體報導，見 §1）。
- 2026 年 8 月的資料刪除／網站下線事件（官方文件，見 §4）。
- credits 不透明與退款爭議（Trustpilot／部落格，次要）。
- 2025 年的獨立評測與社群意見本次**未取得**，不在此填補。

## 11. 對我們的啟發

可做得比 Manus 好的：
1. **所有權與可攜性**：產出物（程式碼、資料、排程定義）預設存放於使用者可控的檔案系統／repo，以 open format 保存；Manus 的教訓是「程式碼可匯出但託管與資料不可攜」，且曾發生平台層級刪除。
2. **成本可預測**：固定成本 + 本機／BYOK 模型，顯示每次任務的預估與上限（硬性 budget cap）；避免 Manus 式不透明 credits 與月底作廢。
3. **隱私與治理風險**：不受單一公司股權／監管事件牽動；資料留在使用者機器。
4. **預設安全**：Manus 提供 "Skip confirmations"；我們可預設保守，並把「可略過確認的範圍」做成細粒度、可審計。

必須對齊的：
- 每 task 隔離的 sandbox、敏感輸入由人接手（Take Over）、重大動作前確認；
- 一鍵發佈 + 自訂網域 + 內建資料庫的「從想法到上線」體驗；
- 持久環境（等同 Cloud Computer）與排程／事件觸發；
- 第三方 connectors（MCP）；
- 即時觀看、停止、以及（Manus 缺少的）可重播與可回滾——這是差異化機會。

## 12. 來源與「未能驗證」

**直接抓取失敗**：manus.im、help.manus.im、trust.manus.im、open.manus.im（DNS 無法解析／proxy 403）。以下為搜尋摘要所引用的 URL：
- 官方：manus.im/docs/introduction/plans；help.manus.im/en/articles/11711111（定價）、11711097（credits 規則）、15392111（Cloud Computer）、15392078（Cloud Computer 計費）、16354507 / 16147831 / 16147895（Service Change）、17190150（Manus 2.0）、11711226（登入資訊管理）、14178443（My Computer）；manus.im/blog/manus-sandbox、manus-cloud-computer、introducing-manus-2-0、manus-schedules、a-note-to-our-users；manus.im/security、manus.im/terms、manus.im/docs/website-builder/{publishing,github-integration,code-control,make-a-copy,custom-domains}；trust.manus.im/faq；open.manus.im/docs。
- 媒體／次要：eWeek（收購）、Fortune／CNBC／Foreign Policy（2026-04 禁令）、TechCrunch 2026-06-13（拆解）、hellochinatech.com（拆解後所有權，付費牆，僅見摘要）、aifundingtracker.com、eesel.ai、usecarly.com、Trustpilot。

**未能驗證**：Manus 2.0 發佈年份與各項效能數字；目前實際股權結構；Cloud Computer 是否額外耗 credits；任務重播／undo／完整稽核日誌；prompt injection 防護；個人記憶機制；Service Change 影響哪類帳號；add-on credits 的現行有效期；2025–2026 的獨立使用者評測。
