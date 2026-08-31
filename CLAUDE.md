# CLAUDE.md — business-idea-generator（商業創意產生器）

「商業創意產生器」——單檔前端工具：使用者輸入「關鍵字1」，系統從內建名詞庫隨機抽出「關鍵字2」，把兩者結合成一個商業創意（創意名稱／描述／5個打造步驟），並以市場需求／收益性／競爭優勢／可擴展性／社會影響五個構面各自 1-5 分評分＋逐步理由＋總平均分數。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

與 `ai-prompt-generator`／`ai-image-prompt-studio` 是姊妹專案，同一套 BYOK 呼叫 LLM 的手法、同一套序號授權鎖整個工具的骨架；評分雙軌邏輯（規則式＋選用AI，前端自算總分）比照 `social-post-grader`，但改為 1-5 分、5 維度、單純算術平均（無 weight）。

## 架構

單一 `index.html`：內嵌 CSS/JS、無外部資源、無建置步驟。視覺主題是深色「創意萌芽」翠綠風格（`--bg #0a140f` + 淡格線背景 + 翠綠色 `--accent #34d399`），與 ai-prompt-generator(青)/ai-image-prompt-studio(洋紅)/Prompt(琥珀)/ai-music-prompt-studio(紫)/social-post-grader(teal) 區隔。

- **`NOUN_BANK`**：關鍵字2的候選名詞庫，扁平陣列，每筆 `{word, category}`，8大類（動物／科技／自然／職業／日常物品／抽象概念／場所／奇幻）各9-10個，共76個，涵蓋「具體↔抽象」光譜以產生有趣的組合張力。`drawNoun()` 用 `Math.random()` 抽取，排除與 `lastDrawnWord` 相同者確保「重新抽一個」必定變化。
- **規則式 fallback（免API金鑰，deterministic）**：`hashString()`（djb2變體字串hash）取代 `Math.random()`，同一組 kw1+kw2 永遠產出相同結果——`generateRuleIdea()` 對創意名稱（`NAME_TEMPLATES`，8種）、描述（`DESC_TEMPLATES`，8種）、5個階段步驟（`STEP_STAGES`：需求驗證→原型/MVP→通路定價→行銷獲客→擴張永續，各3種變體）各用不同 salt 算 hash 選模板；`scoreDimension()` 用 `score = 1 + (hash % 5)`，命中 `CATEGORY_BOOST[category]===dimId` 再加1分並 clamp 1-5，理由文字依分數區間（low1-2/mid3/high4-5，`REASON_TEMPLATES`）挑選。**新增/調整名詞庫或模板池時只需改對應陣列，不需要動抽取或評分邏輯。**
- **AI生成（選用，BYOK）**：`AI_PROVIDERS`／`callLLM()` 與 `ai-prompt-generator/index.html` 同一套實作（Claude 需 `anthropic-dangerous-direct-browser-access` header；OpenAI/OpenRouter 用 Bearer；Gemini 用 `x-goog-api-key`；429/500/503/529 重試3次；180秒逾時）。`buildIdeaPrompt()` 要求LLM只輸出JSON（**故意不讓AI回傳總分**，從源頭避免總分不一致）；`validateAiIdea()` 對 `ideaName`/`description`/`steps`/5個`dimensions`各自獨立驗證（`clampScore()`限制1-5整數、`reason`非空字串），缺哪項就補該項的規則式結果（`missing[]`），不整批放棄，比照 `social-post-grader` 的 `validateAiEvaluation()` 精神。
- **`computeOverall(dims)`**：前端對5個構面分數取算術平均（1位小數），**不信任AI回傳的總分**（prompt本身也沒有要求AI給總分）。`gradeOf(avg)` 換算等第（優異≥4.2／良好≥3.2／普通≥2.2／待加強），對應 `--accent`/`--cyan`/`--amber`/`--red` 四色，SVG圓弧健檢儀表沿用 `social-post-grader` 的 `stroke-dasharray`/`stroke-dashoffset` 技巧，改成 1-5 刻度。
- 狀態存 `localStorage`：`bizIdeaDraft`（`{kw1, kw2}`，reload還原）、`bizIdeaResult`（目前渲染中的結果）、`bizIdeaApiConfig`（`{provider,model,apiKey}`）、`bizIdeaMarquee`（跑馬燈快取）、`bizIdeaSerial`（序號授權）。

## 序號授權（鎖定整個工具，12 個月）

比照 `ai-prompt-generator`／`ai-image-prompt-studio` 的「單一工具、整個鎖住」做法：`#licenseGate` 全螢幕遮罩預設鎖定，驗證通過才加上 `.hidden`；載入時一律對後端即時重驗（不只信任 localStorage 快取），背景每 20 分鐘重驗一次，過期會自動重新鎖住整個頁面。`localStorage` key：`bizIdeaSerial`。

- `Code.gs` — 部署到 Google Sheet 的 Apps Script 原始碼：`doPost` 只做序號驗證＋首次自動啟用，`doGet` 供部署後測試。`VALID_AMOUNT = 12`（月）。部署步驟見 `SETUP-授權伺服器設定.md`。
- **這支後端只做序號驗證，不代理任何付費 API**，也**不處理跑馬燈**（見下）。
- **綁定的 Google Sheet**：使用者指定沿用 <https://docs.google.com/spreadsheets/d/1lhvn0BuoUsn9XZHus-QPXw3guA6Oipgjy0h_qtOd1vw/edit>（表頭與 `ai-image-prompt-studio` 綁定的表一致，但是不同的一份 Sheet；含測試序號 `mark0131`）。
- **部署方式（2026-08-31，本專案首次採用的做法）**：用 `clasp create --parentId <SheetID>` 直接建立綁定腳本專案並 `clasp push`／`clasp deploy`，全程跳過瀏覽器複製貼上與「部署精靈」畫面，比其他姊妹專案的部署流程更快。**唯一差異**：用 API 建立部署會跳過瀏覽器部署精靈附帶觸發的一次性 OAuth 授權，導致部署網址在授權前短暫回應 403「存取遭拒」（已用 curl 實測確認）——使用者已在 Apps Script 編輯器手動執行一次 `doGet` 完成授權（這一步無法由 Claude 代勞，涉及 Google 帳號登入的互動式同意畫面）。**已完成並端對端驗證（2026-08-31）**：用瀏覽器對正式部署網址測試，序號 `mark0131` 正確解鎖顯示「✓ 剩餘 487 天可用」，假序號正確顯示「✗ 查無此授權序號」。`index.html` 的 `LICENSE_CHECK_URL` 已回填部署網址並確認可用，詳見 `SETUP-授權伺服器設定.md`。

## 頂部共用跑馬燈

`#marqueeBar` 內容抓自工作區既有的共用授權伺服器（`https://script.google.com/macros/s/AKfycbwKX0.../exec`，與 `Prompt/index.html`、`ai-prompt-generator`、`ai-video-studio` 系列共用同一個 Google Sheet），做法完全比照 `ai-prompt-generator/index.html` 的獨立跑馬燈邏輯——**跟本工具自己的序號授權後端是兩個互不相干的系統**：頁面載入時直接 POST 一個空序號給共用端點，`localStorage` key `bizIdeaMarquee`，每 20 分鐘背景重抓一次，支援 `[文字](url)` 連結語法。改跑馬燈內容直接編輯共用 Sheet 即可，不需要重新部署任何 Apps Script。

## 隱私與警語

無伺服器端經手使用者資料（序號授權後端除外，只傳送序號本身）；關鍵字、抽出的名詞、產生結果、AI 設定皆只存在使用者瀏覽器的 localStorage。首頁與手冊皆明列使用警語：生成內容僅供發想參考不構成商業決策建議、AI內容需自行查核、請勿輸入真實個資或機密資料、僅供教學與個人使用禁止商業化。修改功能時這些警語需一併檢視是否仍準確。

## 本次未做（後續視需要再處理）

- 未打包可攜式桌面版 exe（需求未明確提及）。
- 未推公開 GitHub repo / 未啟用 GitHub Pages（先本機開發完成、序號授權流程驗證過後再問使用者是否要公開部署）。
- 未加 PWA（加入主畫面）與訪客次數計數器——本次需求只明確要求跑馬燈／創作者資訊／使用警語／RWD，未列入這兩項；若之後要加，比照 `coffee-ig-planner`／`ai-prompt-generator` 的既有做法即可。
- 根目錄 `專案目錄.docx` 尚未加入本專案的列。

## 指令

無建置/測試指令。修改 `index.html` 或 `manual.html` 後直接用瀏覽器開啟驗證，或暫起 `python -m http.server 8801 --directory 行銷內容工具/business-idea-generator` 測完關閉。修改內嵌 `<script>` 後可用以下方式快速檢查語法（把 `<script>...</script>` 內容抽出存成 `.js` 再跑 `node --check`）：

```bash
python -c "
import re
html = open('index.html', encoding='utf-8').read()
open('_check.js','w',encoding='utf-8').write(re.findall(r'<script>(.*?)</script>', html, re.S)[0])
"
node --check _check.js
```

**測試序號授權邏輯前，需先照 `SETUP-授權伺服器設定.md` 部署好 Apps Script 並回填 `LICENSE_CHECK_URL`**，否則會顯示「尚未設定授權伺服器網址」的 fail-closed 錯誤訊息並停留在鎖定畫面；開發階段要測試關鍵字/評分/AI等其他功能，可在瀏覽器 devtools 手動對 `#licenseGate` 加上 `hidden` class 暫時繞過。

驗證 AI 路徑不需要真實金鑰：可在瀏覽器 console 攔截 `window.fetch` 回傳假的 provider 回應格式，確認 `callLLM → extractJsonObject → validateAiIdea → renderResult` 整條管線正確（含單一構面驗證失敗時的單點 fallback），測完記得還原 `window.fetch`。
