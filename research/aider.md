# Aider 研究報告

檢查日期：2026-10-25。註：aider.chat 在本環境 DNS 無法解析（ENOTFOUND），因此「docs」內容改取自 repo 內的原始檔 `aider/website/docs/*.md`（raw.githubusercontent.com，屬官方一手來源）。GitHub API（api.github.com）回 403，未能取得精確 API 數據。

## 1. 基本資料
- Repo：github.com/Aider-AI/aider；License：Apache-2.0（已驗證）；主要語言：Python（README/安裝文件為 pip 套件 aider-chat，GitHub 語言欄在抓取結果中未顯示，Python 為由安裝方式與原始碼檔名 .py 所作的推論，但原始碼確為 .py 已驗證）。
- Stars 約 49.7k、forks 5.1k、watchers 266、總 commits 13,138（github.com 頁面，已驗證）。
- 維護者：Paul Gauthier（leaderboard 頁署名；HISTORY 長期為其個人專案）。Bus factor 偏高（實質單一核心維護者）。
- 發布節奏：releases 頁最新為 v0.86.0（"09 Aug"，頁面未顯示年份；依 0.85.0=27 Jun、0.84.0=30 May、0.83.0=09 May、0.82.0=14 Apr 等推論為 2025 年）。0.77→0.86 約 3-5 週一版，之後無新 release 可見（我們於 2026-10 查證時最新仍是 v0.86.0，即約一年多沒有新版；推論，年份未驗證）。HISTORY.md 頂部亦是 v0.86.1 之後的 main branch 區段。
- main 分支仍有活動：最近 commits 為 2026-05-22（擴充 ANTHROPIC_MODELS 清單，PR #5173）、2026-05-15（新增 bash tree-sitter tags）、2026-04-24/25（gpt-5.5 model settings）。即「主幹有零星、多為模型設定/社群 PR 的更新，但沒有新 release」。
- 維護疑慮訊號：issue #4613「Where is Paul?」（2025-11-15 關閉，19 則留言，內文未讀）、#4751「Thoughts and questions about the future direction of aider」（2026-01-07 開啟，仍 open）。內容未讀，只能說存在此類討論。
- v0.86.0 HISTORY 註記 aider 自己寫了 88% 的程式碼（v0.85.0 為 21%）。

## 2. 架構
- 核心為「聊天迴圈 + Coder 類別」：每種 edit format 對應一個 Coder 子類別（原始碼 aider/coders/，如 editblock、wholefile、udiff、architect 等；目錄命名由記憶推論，未逐一開檔驗證）。使用者訊息 → 組 prompt（system + repo map + 聊天中檔案 + 歷史）→ LLM 回覆 → 解析回覆中的編輯區塊 → 套用到檔案 → git commit → 可選 lint/test 並自動修復。
- 與 tool-calling agent 的差異：LLM 輸出「文字格式的編輯」（search/replace 區塊等），由 aider 自行解析套用，而非呼叫 function/tool 來讀檔、跑指令。檔案需由使用者 `/add` 進聊天（或 aider 建議後使用者同意）；模型無自主「探索檔案 → 執行 → 觀察」迴圈。
- 仍屬「單回合 + 人類在迴圈」：lint/test 失敗時會自動把輸出餵回去修（有限的反饋迴圈），但不是通用 agent loop。
- 變化驗證：HISTORY.md 與文件搜尋未見 MCP、agent mode、tool calling 條目（HISTORY 前 8 版與全頁均無）。issue #4506「Add native MCP server and Agent Mode support to Aider CLI (as in AiderDesk)」（2025-09-09）仍 open，表示官方尚未內建；AiderDesk 是第三方分叉/桌面應用（我們僅由該 issue 標題得知，未驗證）。
- 模式（modes.md）：code、ask（不改檔）、architect、help；另有 `/context` 指令（commands.md 列出，功能細節未驗證）。官方建議 ask → code 交替。

## 3. Edit formats（edit-formats.md，已驗證）
- whole：回傳整檔，最簡單但慢且貴。
- diff：search/replace 區塊（類似 git merge conflict 標記），檔名在 fence 之前；只傳改動，高效。
- diff-fenced：檔名放進 fence 內，主要為 Gemini 系列不遵守 diff 圍欄慣例而設。
- udiff：簡化的 unified diff，原為對抗 GPT-4 Turbo「lazy coding」（用省略註解取代程式碼）。
- editor-diff / editor-whole：architect 模式中 editor 模型用的精簡版 prompt（`--editor-edit-format`）。
- aider 會依模型選預設格式，可用 `--edit-format` 覆寫。存在原因：不同模型遵循格式的能力不同，格式直接影響成功率與 token 成本。
- Leaderboard（leaderboards/index.md）：polyglot benchmark，225 個 Exercism 難題，涵蓋 C++/Go/Java/JavaScript/Python/Rust；欄位含 percent correct（主排序）、cost、command、correct edit format 百分比、edit format。頁面最後更新 2025-11-20（Paul Gauthier）。頂部名次/分數在抓取內容中未渲染，未能驗證。

## 4. Repo map（repomap.md + repomap.py，已驗證）
- 用 tree-sitter 依語言的 `.scm` tags query 擷取 `name.definition.*`（def）與 `name.reference.*`（ref）；只有 def 無 ref 的語言用 Pygments 補 ref。2026-05 有新增 bash tags（commit）。
- 建圖：networkx MultiDiGraph，由引用檔指向定義檔；PageRank 加 personalization（聊天中檔案、被提及的檔名/識別字得分 100/len(fnames)）。邊權重：被對話提及的識別字 ×10；長度≥8 的 snake/kebab/camelCase ×10；底線開頭 ×0.1；定義於超過 5 個檔 ×0.1；來自聊天中檔案的邊 ×50；引用次數取平方根抑制。
- 預算：`--map-tokens` 預設 1k（文件）；無檔案在聊天中時可放大（程式：`map_mul_no_files` 預設 8，並受 context window−4096 限制）。以二分搜尋決定納入多少排名 tags，容差 15%。
- 快取：diskcache 存於 repo 根目錄 `.aider.tags.cache.v3/v4`，依 mtime 重新解析；`--map-refresh` 預設 auto（上次建圖超過 1 秒才快取整張圖）。
- 非 embedding / 非向量檢索；純靜態符號圖。

## 5. Git 整合（git.md + commands.py，已驗證）
- 預設 `--auto-commits` 開啟：每次 aider 編輯後自動 commit；目錄非 git repo 時會提議建立。
- Dirty files：編輯前若檔案有未提交變更，先單獨 commit 這些（`--no-dirty-commits` 可關），讓人與 AI 的變更分離。
- 署名：aider 作者的變更，git author 與 committer 名稱加 "(aider)"；僅提交既有 dirty 變更時只加 committer。可用 `--no-attribute-author/committer`、`--attribute-commit-message-author/committer`（訊息前綴 "aider: "）、`--attribute-co-authored-by`（加 Co-authored-by trailer）。
- commit message 由 weak model 產生，預設 Conventional Commits，可用 `--commit-prompt` 改。預設跳過 pre-commit hooks（`--git-commit-verify` 預設 False）。
- `/undo` 限制（程式碼）：只能撤銷 HEAD、且必須是「本次聊天 session 中 aider 所做」的 commit；首個 commit、merge commit 不可；相關檔案有未提交變更需先 stash；若 origin/<branch> 與本地 HEAD 相同（已 push）拒絕；實作為各檔 `git checkout HEAD~1` 後 `git reset --soft HEAD~1`。其他：`/diff`、`/commit`、`/git`；`--no-git` 可完全停用（官方警告需自行備份）。
- 限制：只有一層簡便 undo（連續 /undo 可逐步退，但受上述條件約束；多步行為未驗證）。

## 6. 權限與安全
- 無沙箱。編輯權限來自使用者把檔案加入聊天；模型只能經 edit format 改這些檔（新檔案建立另有流程，細節未驗證）。
- `/run`（別名 `!`）：由使用者執行 shell 指令，輸出是否加入聊天需 `confirm_ask` 確認；`/test`：無確認提示，非零退出碼才把輸出加入聊天並（搭配 `--auto-test`）觸發自動修復。
- `--suggest-shell-commands` 預設 True：模型可建議 shell 指令，由使用者決定是否執行（確認行為由預設值與設計推論，詳細提示流程未驗證）。
- lint 預設自動執行（`--no-auto-lint` 關閉）；`--lint-cmd`、`--test-cmd`、`--auto-test` 可設定。
- 保護機制主要是 git：自動 commit + `/undo` + 預設不跳過的 dirty 分離。沒有權限分級、沒有 sandbox、沒有 diff 預先核准流程（issue #649「force the AI to ask the user to confirm each change before doing it」有 39 個 +1，仍 open）。
- `--check-update` 預設 True、analytics 為抽樣（描述寫 random，細節未驗證）。

## 7. 模型與成本
- 透過 litellm 支援大量雲端與本地模型（README 提及 Claude、DeepSeek、OpenAI o 系列、GPT-4o 與本地模型；「litellm」為原始碼依賴，我們未在本次抓取中直接驗證其頁面）。HISTORY/commits 顯示持續更新模型設定（2026-04 gpt-5.5、2026-05 Anthropic 模型清單）。
- Architect/editor 分離：architect（`--model`）提出解法，editor（`--editor-model`，預設依主模型挑）轉成編輯；每個請求兩次 LLM 呼叫，成本與延遲上升；`--auto-accept-architect` 預設 True。適合推理強但編輯差的模型（文件舉 o1）。另有 weak model 用於 commit message 與摘要。
- Prompt caching：僅限 Anthropic（Sonnet/Haiku）與 DeepSeek Chat；快取 system prompt、read-only 檔、repo map、可編輯檔；`--cache-prompts`；串流時不顯示 cache 統計與成本（需 `--no-stream`）；`--cache-keepalive-pings N` 每 5 分鐘 ping。
- 成本顯示：聊天中提供 token/cost 回報、`/tokens` 看 context 用量（命令已驗證；回報格式細節未驗證）。
- 其他：`/reasoning-effort`、`/think-tokens`、`/copy-context`（供貼到網頁版聊天）。

## 8. 上手體驗
- 安裝（install.md）：`aider-install`（獨立 Python 環境）、curl/PowerShell 一行安裝（會裝 Python 3.12）、uv、pipx、pip；官方警告不要用系統套件管理員。pip/pipx 支援 Python 3.9–3.12；uv 可用現有 3.8–3.13 或另裝 3.12。
- 首次執行與 API key：該頁未涵蓋，需看其他文件頁（未取得，未驗證）。
- 推論的新手痛點：需先有 API key、懂 git 與終端機、需手動 `/add` 檔案、不知道選哪個模型/edit format、Python 環境問題（版本上限 3.12 為文件所述，與新版 Python 相容是風險）。

## 9. 擴充與自動化
- 腳本：`--message/-m`、`--message-file/-f`、`--yes`、`--dry-run`、`--commit`；Python API（`Coder.create` / `coder.run`）官方明言「未正式支援、可能不相容更動」。
- Watch-files：`--watch-files` 監看 `AI`、`AI!`（執行修改）、`AI?`（提問）註解，可搭配任何 IDE；加 `#AI` 可把檔案加入聊天。
- 語音 `/voice`、`/web` 抓網頁轉 markdown、圖片作為上下文、`/paste`、`/copy-context` 等（命令清單已驗證，實際行為未逐一測）。
- Conventions：`CONVENTIONS.md` 以 `--read` / `/read` 載入（read-only，可被 cache），或 `.aider.conf.yml` 的 `read:`。
- MCP：未內建（見第 2 節）；#2525（27 +1）、#3314（25 +1）、#4506、#3658（Aider 當 MCP server）皆 open。第三方 proxy 討論 #5192。

## 10. 優缺點與痛點
優點：git-native 安全網、repo map 讓大專案可用、模型無關、edit format 與 benchmark 有科學方法、低成本（無額外 agent 迴圈浪費）、watch mode 與 IDE 無關。
缺點：無自主 agent 能力（探索檔案、跑指令迭代）；無 MCP；無 sandbox 或權限層；release 停滯、核心維護者單一。
新手痛點（推論 + issue 佐證）：終端機/git/API key 門檻；需手動管理檔案上下文；Python 環境。
專業使用者痛點：#3362「Inspiration From Claude Code」（50 +1）顯示對 agent 化的期待；#649 要求逐項變更確認；#68 要求 VS Code extension（36 +1）；#2227 GitHub Copilot provider（214 +1，最高）；#3603 repo map 納入相依套件的問題（17 +1）；MCP 需求。

## 11. 對我們的啟發
- 值得抄：(1) git 自動 commit + dirty 分離 + 簡單 undo——對 vibe coder 極有價值，但應補強 /undo 的限制（可多步、可跨 session、已 push 也能安全 revert）。(2) repo map（tree-sitter + PageRank + token budget 二分搜尋 + 快取）。(3) edit-format benchmarking 與按模型挑格式的機制，及公開 leaderboard。(4) architect/editor 雙模型。(5) AI 註解 watch mode 對新手友善。(6) CONVENTIONS 檔 + prompt caching 組合。
- 避免：缺少 sandbox/權限分級；新手必須自己 `/add` 檔案；安裝與 API key 設定繁瑣；Python API 不穩定；單一維護者風險；無 MCP 導致生態落後。
- 差異化機會：把 aider 的 git 安全網 + repo map 與 agent loop、MCP、視覺化 diff 審核、引導式 onboarding 結合。

## 12. 來源與未能驗證
來源（皆一手）：github.com/Aider-AI/aider（首頁、commits/main、releases、issues 搜尋頁）；raw.githubusercontent.com/Aider-AI/aider/main/ 之 HISTORY.md、aider/repomap.py、aider/commands.py、aider/website/docs/{more/edit-formats,repomap,git,usage/modes,usage/commands,usage/lint-test,usage/watch,usage/caching,usage/conventions,scripting,install,config/options,leaderboards/index}.md。
未能驗證：aider.chat 網站本身（DNS 失敗）；GitHub API（403）因此無精確 release 年份、貢獻者數、open issue 數；leaderboard 實際名次與分數；blog 的 repo map 文章；首次執行與 API key 文件；coder 類別清單細節；shell 指令建議的確認流程；多步 /undo 行為；issue 內文（#4613、#4751 等只讀到標題）；litellm 依賴頁；cost 顯示格式；analytics 實際比例；v0.86.0 之後是否有 release（僅由 releases 頁推論）。
