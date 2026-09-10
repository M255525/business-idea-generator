# CLAUDE.md — business-idea-generator（商業創意產生器）

「商業創意產生器」——單檔前端工具：使用者輸入「關鍵字1」，系統從內建名詞庫隨機抽出「關鍵字2」，把兩者結合成一個商業創意（創意名稱／描述／5個打造步驟），並以市場需求／收益性／競爭優勢／可擴展性／社會影響五個構面各自 1-5 分評分＋逐步理由＋總平均分數。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

與 `ai-prompt-generator`／`ai-image-prompt-studio` 是姊妹專案，同一套 BYOK 呼叫 LLM 的手法、同一套序號授權鎖整個工具的骨架；評分雙軌邏輯（規則式＋選用AI，前端自算總分）比照 `social-post-grader`，但改為 1-5 分、5 維度、單純算術平均（無 weight）。

## 架構

單一 `index.html`：內嵌 CSS/JS、無外部資源、無建置步驟。視覺主題是深色「創意萌芽」翠綠風格（`--bg #0a140f` + 淡格線背景 + 翠綠色 `--accent #34d399`），與 ai-prompt-generator(青)/ai-image-prompt-studio(洋紅)/Prompt(琥珀)/ai-music-prompt-studio(紫)/social-post-grader(teal) 區隔。

**底色主題選擇器（2026-09-10 新增）**：topbar 右上角「底色」四個圓形色票（`#themePicker`），依使用者要求必含一個淺色主題。全部靠重新定義既有 CSS variables 實現，不新增另一套樣式表：
- `green`（預設，深翠綠，原始配色）／`violet`（深紫夜）／`slate`（深藍）／`light`（**淺色**，白底＋深綠文字）。
- 選擇存在 `localStorage['bizIdeaTheme']`；`<head>` 最前面有一段極早期 inline script 在畫面繪製前就讀出並設定 `<html data-theme="...">`，避免換頁時先閃一下預設深色再跳成使用者選的主題（flash of wrong theme）。
- **`--accent`／`--violet` 分成「文字/邊框用」與「純色按鈕底用」兩個變數**（新增 `--accent-fill`／`--violet-fill`）：因為同一顏色不能同時滿足「淺色主題下當文字要夠深才能在白底上看清楚」與「當 `.btn.primary`／`.btn.violet` 純色按鈕底、要夠亮才能配上寫死的深色文字（`#04120c`／`#1b1330`）」這兩個互斥的對比度需求——`light` 主題把 `--accent` 調暗（`#0e9d63`，給文字/邊框用）但 `--accent-fill` 仍保留原本鮮綠 `#34d399`（給按鈕底用），`--violet`／`--violet-fill` 同理。深色主題（`green`/`violet`/`slate`）這兩個變數維持相同值，行為不變。
- 按鈕 hover 效果從寫死的淺色 hex 改成 `filter:brightness(1.1)`，這樣任何主題的 hover 都會自動變亮而不用逐主題再寫一次 hover 色。
- `--grid-line`（body 背景格線的顏色）、`--cyan`/`--amber`/`--red`（評分等第顏色）也各主題各自覆寫，確保在白底上仍有足夠對比（例如 `light` 主題的 `--amber` 用較深的 `#a35b09` 取代原本較亮的 `#fbbf24`）。
- 已用瀏覽器實測四個主題皆正確切換（含已產生結果的評分卡片/量表/進度條），且 reload 後主題正確保留、無閃爍。

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

## GitHub 與線上部署

已推公開 GitHub repo：<https://github.com/M255525/business-idea-generator>，已用 `.github/workflows/deploy-pages.yml`（Actions 部署模式，`gh api repos/M255525/business-idea-generator/pages -f build_type=workflow` 啟用，比照 `workspace-git-repos` 記載的「不要用 legacy branch-source」慣例）啟用 GitHub Pages：<https://m255525.github.io/business-idea-generator/>（2026-08-31 上線，已用 curl 與瀏覽器對正式網址驗證頁面與序號授權皆正常運作）。

## 加入主畫面（PWA，2026-08-31 新增）

比照 `ai-prompt-generator`／`coffee-ig-planner` 的做法：`manifest.json`＋`icons/`（翠綠 `#34d399` 底＋白色「創」字，PIL＋`msjhbd.ttc` 產生，192/512/maskable-512/apple-touch-icon 四種尺寸，產生腳本未進 repo，比照工作區慣例）＋`service-worker.js`（network-first＋同源快取備援，`fetch(request,{cache:'reload'})` 這個細節必須保留，否則 GitHub Pages 的 HTTP 快取會讓 network-first 失效）。頁尾 `.footer-meta` 新增「📲 加入主畫面」按鈕（`#installBtn`）＋獨立 IIFE（含 iOS/macOS Safari 判斷、自帶 `notify()` 不依賴外部 `showToast()`，逐字沿用 `ai-prompt-generator` 已修過 bug 的版本，避免重蹈「按鈕沒反應」的舊坑）。`<head>` 同步補上 `manifest` link／`apple-touch-icon`／`mobile-web-app-capable`／`apple-mobile-web-app-*` 系列 meta。已用瀏覽器實測：Service Worker 成功註冊（`getRegistrations()` 回傳1筆），且 Chrome 判定頁面符合安裝條件並觸發真實 `beforeinstallprompt`（測試時用合成 click 觸發 `.prompt()` 因缺少真實使用者手勢而拋出 `NotAllowedError`，這是測試方法本身的限制不是功能缺陷——真人點擊時會正常運作，比照 `coffee-ig-planner` 已記錄過的同一種測試限制）。

## 訪客次數計數器（2026-08-31 新增）

頁尾 `.footer-meta` 加了 `visitor-badge.laobi.icu` 的 SVG badge（`<img>` 直接嵌入，`page_id=m255525.business-idea-generator`，免金鑰免後端），做法比照 `SocialPost`／`ai-prompt-generator` 已驗證過的模式，已用瀏覽器實測圖片成功載入。

## 匯出 TXT／PDF（2026-08-31 新增）

- **TXT**：`buildTextReport(kw1, kw2, result)` 組出純文字報告（創意名稱/描述/步驟/五構面評分理由/總平均分），`downloadText()`＋`sanitizeFilename()` 用 Blob + 隱藏 `<a download>` 觸發下載，逐字比照 `coffee-ig-planner` 的做法搬過來。已用 Playwright `waitForEvent('download')` 攔截驗證檔名（`關鍵字1×關鍵字2-商業創意.txt`）與內容皆正確、UTF-8 編碼無亂碼。
- **PDF**：比照 `restaurant-feasibility-calculator` 的「獨立靜態報表」路線（**非**使用 jsPDF/html2pdf 等函式庫，純瀏覽器原生 `window.print()`）——`#printReportRoot`（畫面上永遠 `display:none`）平常不可見，按下「🖨️ 匯出 PDF」時 `buildPrintReport()` 把結果組成一段 HTML（所有動態文字皆過 `escapeHtml()`）塞進去，再呼叫 `window.print()`；`@media print{ body>*{display:none!important} #printReportRoot{display:block!important} }` 全域隱藏法，比逐一排查隱藏 UI 元件可靠、不會有分頁空白頁問題（`restaurant-feasibility-calculator` CLAUDE.md 記載過的已知坑：改用 `display:none` 而非 `visibility:hidden` 才能避免）。已用 Playwright `page.emulateMedia({media:'print'})` + 截圖驗證：列印版面正確顯示白底黑字報告（含表格化的五構面評分），其餘 UI／深色主題／跑馬燈皆正確隱藏；`body{background:#fff!important;padding:0!important}` 這行是必要的——不加的話 `body` 本身（非其子元素）仍會保留深色格線背景與 `padding-top:30px`，在列印版面頂端留下一段跟報告內容不搭的深色空白。

**PDF 浮水印（2026-08-31 應使用者要求追加）**：`#pdfWatermark`（`<img id="wmImg">`）逐字比照 `restaurant-feasibility-calculator` 的做法——內嵌 base64 data URI（不是 CSS `background-image`，因為會被瀏覽器「列印背景圖形」選項預設擋掉，`<img>` 是內容一定會印出來）、`position:fixed;opacity:.11` 讓每頁都重複出現。**圖檔直接沿用工作區既有的已處理版本**：`資料儀表板/IPA_Kano/watermark-source.png`（480×297、已去背 RGBA，「馬克老師 AI・工具・學習・成長」品牌圖示，跟 `IPA_Kano`／`restaurant-feasibility-calculator` 用的是同一張），複製到本專案根目錄 `watermark-source.png` 後用 Python 腳本 base64 編碼、字串替換塞進 `index.html` 的 `WATERMARK_DATA_URI` 常數（約151KB，未經過對話視窗，避免灌爆 token）。**TXT 匯出**因為是純文字無法放圖片，改在報告末尾加一行文字署名「馬克老師｜AI・工具・學習・成長」作為對應的文字版本。已用 Playwright 截圖確認 PDF 浮水印正確置中淡出顯示、不影響內容可讀性；TXT 下載內容確認含署名行。

## 本次未做（後續視需要再處理）

- 未打包可攜式桌面版 exe（需求未明確提及）。
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
