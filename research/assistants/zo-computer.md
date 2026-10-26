# Zo Computer 研究筆記（文件層級，非實測）

> **方法與限制（必讀）**：本檔案作者無法使用產品。官方站點 zo.computer／www.zo.computer 以 WebFetch 直接抓取全部失敗（`getaddrinfo ENOTFOUND`；curl 經 proxy 回 `CONNECT 403`）；GitHub `SubstrateLabs/zo-computer` 回 404。因此下列「官方」內容均來自 WebSearch 工具回傳的**搜尋摘要（對官方頁面的二手轉述）**，並附官方 URL；我沒有親讀原頁全文。標示：【官方（經摘要）】、【第三方／次要】、【競品／廠商發表，僅作線索】。**Zo 自己的頁面彼此有出入**（見 §1、§8），採用前請回官方核對。
>
> 查核日期：2026-10-26。

## 1. 基本資料

- **公司**：Substrate，現以 Zo Computer 為品牌（Zo Computer 是 Substrate 目前的名稱）；共同創辦人 Rob Cheung（CEO，前 Substack）與 Ben Guo（前 Stripe Terminal）。【第三方公司資料：ai.engineer、Tracxn、PitchBook 摘要】
- **注意同名混淆**：另有一家做微影設備的「Substrate」（James Proud），與 Zo 無關；微軟舊聊天機器人「Zo」亦無關。
- **所有權／狀態**：獨立新創，營運中（官方有至 2026-09 的 Product Updates）。投資人（第三方）：Lightspeed（領投早期 800 萬美元，針對原 Substrate 平台）、South Park Commons、Craft Ventures 與天使（Guillermo Rauch、Immad Akhund）。創立年份各來源不一（2020／2023），資金細節**未驗證**。未見被收購或停運的報導。
- **可用性**：雲端託管服務（managed），**不是自架**。Web、SMS、email、Telegram、SSH 連線。【官方（經摘要）】
- **定價模型**：訂閱（含運算資源）+ AI credits（另計，可 BYOK）。【官方 pricing／billing 頁（經摘要），兩頁互相矛盾】
  - Free Trial：pricing 頁稱 14 天，之後僅「recovery only」；billing 文件另一版本稱免費層為每日免費模型額度。**哪個為現況：未驗證**。
  - Basic US$18/月（含 US$10 AI）、Pro US$64/月（含 US$40）、Ultra US$200/月（含 US$100）。
  - Teams：公司電腦 US$200/月 + 每位成員 US$18。
  - 資源規格兩頁不一致（pricing 頁 Basic 2 vCPU/8GB；billing 文件 Basic 4 CPU/32GB、Pro 16/128GB、Ultra 64/512GB）——**無法判定哪個正確**。

## 2. 定位與使用場景

- 官方定位：「個人雲端電腦，由你與 AI 共同操作」；AI 對一台專屬雲端電腦有完整存取，可瀏覽網頁、寫並執行程式、託管服務。【官方（經摘要）】
- 範例：用簡訊交辦工作、排程 agents、託管個人網站（zo.space）、spreadsheet 轉可查詢資料庫、「second brain」、跑 Hermes Agent／OpenClaw／Claude Code／Codex 等第三方 agent（官方有教學頁）、Teams 共享工作區與核准流程。

## 3. 執行環境

- **跑在哪**：Zo 管理的雲端容器；安全頁列出的 sub-processors 包含 Modal（雲端運算）、Vercel AI Gateway（多家模型推論）、Postmark（email，保留 45 天標頭、不留內容）、Telnyx（簡訊，保留 10 天）。
- **持久性**：是——使用者得到一台常駐的 Linux 電腦與完整檔案系統（官方）；背景服務（HTTP、API、TCP、資料庫）開機自啟、當掉重啟。試用／低階方案「閒置時休眠」。有第三方評測者稱存 2GB 資料 72 小時後仍完好（次要、單一軼事）。
- **可觸及範圍**：網路、內建 agentic browser（官方稱在你的伺服器上的隔離環境執行，不使用你個人瀏覽器的登入狀態）、app 連線（Gmail 等，inline Connect widget）、SSH（需預先登錄公鑰）、各種 AI 供應商。
- 付款停止後：運算與一般存取結束，檔案仍可取回（見 §5、§7）。

## 4. 權限與安全模型

（皆為**廠商宣稱**，無法獨立驗證）
- **隔離論述**：官方稱讓 AI 在專屬雲端電腦上操作「比在你自己的機器上跑 agent 更安全，但並非沒有風險」；安全頁明確承認風險。
- **核准**：沒有找到「每個工具呼叫都需確認」的全域設定。核准主要靠 **Rules**（可寫成條件式，例如「未經我同意絕不寄信」）與 **persona 的工具權限範圍（tool scopes／presets）**；官方教學建議「Zo 起草不寄出，對外動作保留人工關卡，無人值守的 persona 給不能對外傳訊的 scope」。也就是說，**安全性在相當程度上交給使用者自己用自然語言規則設定**，不是強制預設。
- **憑證處理**：搜尋**未找到** API key／secrets 的儲存機制文件（未驗證）。比較頁僅宣稱「你的登入留在你的 Zo 環境內」，屬行銷語句。
- **Prompt injection**：安全頁只說「對 prompt injection 與 LLM 安全研究有興趣，歡迎回報，但因 LLM 本質，這類漏洞通常不納入賞金」。**未描述任何防禦措施**，等同間接承認無法完全防範。
- **資料保留與使用**：官方稱不賣資料、不放廣告、不以使用者作品訓練模型；定期替檔案系統做快照；sub-processors 標示「無資料存取／部分／儲存」，「部分」指檔案片段或整檔可能被選入推論上下文。
- **廠商自承限制**：自架被描述為長期願景而非現況；agent 權限廣，連接 email／行事曆時有風險（安全頁與第三方評測皆提及）。

## 5. 復原與可稽核性

- **快照／還原**：官方稱定期快照，可還原先前版本；整台電腦可作為 container image 提供下載（安全頁／FAQ）。
- **停止後取回**：試用結束或停止後，owner 可啟動最多七次、每次一小時的 recovery session 下載工作區資料；失敗時寄信 help@zocomputer.com 由客服準備 ZIP。【官方 billing（經摘要）】
- **觀看／停止／重播／undo**：agent 對話可見；排程 agent 額度用完時會通知「已略過」而非靜默失敗（官方）。可逐步重播或單步 undo 的功能：**未找到**。計費層面，Stripe 事件失敗時會保留使用記錄作為稽核證據、不自動重送扣款（官方 billing）。

## 6. 記憶、排程與主動行為

- **記憶**：官方稱 persistent memory，偏好、專案名稱、人物會跨對話保留；另有檔案系統本身作為知識庫。第三方評測者有人抱怨「對我如何工作幾乎沒有脈絡」（次要）。
- **Rules**：條件式規則，決定何時／如何回應。
- **Personas**：自訂 agent 設定（指示、語氣、偏好模型、工具範圍），可跨 chat／SMS／email／Telegram；有公開 persona 可分享（zo.computer/pub/persona/…）。
- **排程 agents**：用自然語言描述週期工作，agent 依排程帶檔案、瀏覽器、app 連線執行；可設定停用 AI credit 支出（BYOK 與免費模型仍可用）。
- **主動行為**：透過排程與 rules 實現；自發性建議未找到文件。

## 7. 能否替使用者做出東西（關鍵軸）

- **網站與服務**：Zo Sites——每站有私有開發 URL 與可選公開 URL，publish 時做 production build；可執行 React app、API server、資料庫、長駐程序（官方）；zo.space 的 API routes 以 Hono 建立；zo.pub 可發佈內容。
- **自訂網域**：僅子網域（不支援 apex），CNAME 指向 `cname.zocomputer.io`，DNS 設定後約 15 分鐘；付費方案才有（Basic 3、Pro 5、Ultra 10 個；試用 0）。服務數上限（Basic 最多 5 個）。【官方（經摘要）】
- **使用者最終擁有什麼**：
  - 一台以**開放格式、可攜 container image** 保存的檔案系統與程式碼——可取回（還原 session／ZIP／container image 請求）。這是明顯的可攜性優點。【官方宣稱】
  - 但**託管的執行環境、agent 本身、帳號下的服務與網域路由仍依賴 Zo**（第三方評測亦指出：檔案可攜，托管 agent 與服務依賴 Zo）。Zo 應用程式本身**不能自行運行**（官方稱為長期願景）。取回 container image 是否能在別處無痛重現其服務與網域設定：**未驗證**。
  - 腳本與排程自動化：以檔案 + 排程 agent 定義存在你的 Zo 內；排程由 Zo 平台觸發。
- **品質問題（次要）**：一位開發者回報 agent 自認完成但網站其實壞掉，需不斷提醒重新 publish。

## 8. 成本結構

- 兩層計費：(1) 訂閱買「電腦」；(2) AI credits 買模型用量。**Credits 與外部 AI 供應商不會延長電腦存取**；試用期後即使有 credits 或自帶 key，也要付費方案才恢復一般電腦存取。可買 credits、自動補值；可關閉 AI 花費；可 BYOK（Claude、ChatGPT、Gemini）或把 Claude Code／Codex 訂閱當 provider。
- 相較按任務計價的 agent，Zo 的月費是固定的（含一定 AI 額度），超出額度才加買——成本可預測性較佳，但**超額行為與細節未能完整驗證**。
- 意外成本抱怨：本次搜尋**未找到**可靠的 Zo 計費投訴；兩份官方頁面對免費層與規格的矛盾本身即是風險訊號。

## 9. 上手體驗（非技術使用者）

- 官方訴求：用簡訊／聊天指揮；提供大量教學、persona 範本。
- 第三方評測稱「不像一般 SaaS 那樣即開即用」，需要整理電腦與 agent 權限，初期偏複雜（次要，且多為聯盟式／部分 AI 生成的評論網站，可靠度有限）。Product Hunt 有評論頁（未讀）。

## 10. 已知問題與外部評價（次要，標示）

- 可靠性與完成度：「有趣但尚未達商用」、錯誤與緩慢（lowcode.agency 等）。
- 發佈脆弱（個人部落格 jplhomer.org 類開發者回報）。
- 社群資源與文件有限；agent 權限廣的安全曝險；對 Zo 平台的依賴。
- 競品文章（Vellum 稱其為「附帶 AI 的伺服器」、zo.computer 自己的 vs-OpenClaw/Comet 比較頁）**屬競品／廠商自發表，僅供線索**，本檔不引用其結論。
- Reddit／Hacker News 的原始討論：搜尋未取得。

## 11. 對我們的啟發

可做得比 Zo 好的：
1. **真正自架**：Zo 的管理式模式意味資料與服務在 Zo／Modal／Vercel AI Gateway 上；開源自架可讓「部分資料進入推論上下文」的風險降到使用者自選的模型與主機。
2. **可攜性升級**：Zo 已證明「開放格式 + container image 匯出」是賣點；我們可以讓整個環境（檔案、服務定義、排程、persona/rules）本來就是使用者的 repo／compose 檔，而非要求客服寄 ZIP。
3. **安全預設**：Zo 的核准依賴使用者自己寫 Rules，prompt-injection 無防禦說明；我們可預設「對外動作需核准」、最小權限 scope、機密與 agent 分離，並提供明確的威脅模型文件。
4. **成本透明**：固定自有硬體 + BYOK，並在 credits 用完時明確行為（Zo 已做到「略過並通知」，應對齊）。

必須對齊的：
- 常駐 Linux 環境、背景服務自啟／重啟、快照與還原；
- 簡訊／email／Telegram 多通道、persona + rules + 排程 agents；
- 發佈網站 + 自訂網域（至少子網域）的簡單流程；
- 退出機制（停止付款後仍可取回資料）。

## 12. 來源與「未能驗證」

**直接抓取失敗**：zo.computer、www.zo.computer（DNS／proxy 403）；github.com/SubstrateLabs/zo-computer（404）。以下為搜尋摘要引用的 URL：
- 官方：zo.computer/pricing、/faq、/docs/billing、/guide/billing、/docs/information/security（亦見 /reference/information/security）、/docs/information/privacy-policy、/websites、/app/hosting、/app/rules、/guide/custom-domains、/guide/zo-pub、/guide/ssh-zo、/docs/tools、/updates、/teams、/your-zo、/comparisons/*（比較頁屬廠商自發表）。
- 第三方公司資料：ai.engineer/orgs/substrate、venturebeat（Substrate 800 萬美元）、PitchBook、Tracxn。
- 次要評測：lowcode.agency、swanbase.co、jplhomer.org、agent-finder.co、tools.stoa.agency、Product Hunt；競品：vellum.ai。

**未能驗證**：現行免費層（14 天試用或每日免費額度）；各方案實際 CPU/RAM；取消訂閱後電腦與檔案的保留期；secrets／API key 儲存機制；核准機制的預設值；container image 在別處重現服務的可行性；融資與成立年份；是否有自家開源程式碼（GitHub repo 404）；真實用戶的計費與穩定性投訴（Reddit／HN 未取得）。
