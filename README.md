# AGENTS.md — 學術論文中文翻譯與重點整理網站開發規範

## 1. 專案願景與核心定位
本專案旨在建立一個**零後端依賴、雙擊即可在瀏覽器運行**的純前端學術論文閱讀工具。使用者上傳英文 PDF 論文後，系統透過瀏覽器端解析文本，並呼叫 Google Gemini API 自動生成**結構化繁體中文重點導讀**與**高品質學術全文翻譯（支援中英雙語對照）**。

---

## 2. 技術架構與單一檔案限制 (Zero-Build Single File)
1. **單一檔案原則：** 所有 HTML 結構、CSS 樣式與 JavaScript 邏輯**必須封裝在單一 `index.html` 檔案中**。嚴禁拆分出獨立的 `.css`、`.js` 檔案或要求使用 `npm` / 建置工具。
2. **第三方套件引入（僅限穩定 CDN）：**
   * **PDF 解析：** `PDF.js`（需正確設定 `pdfjsLib.GlobalWorkerOptions.workerSrc`）。
   * **Markdown 渲染：** `Marked.js`（將 AI 回傳的結構化內容與表格渲染為乾淨的 HTML）。
   * **數學與化學公式渲染：** `KaTeX`（包含 auto-render 擴充套件，確保論文中的 `$ ... $` 與 `$$ ... $$` LaTeX 公式正確排版）。
3. **本機資料持久化 (`LocalStorage`)：**
   * 使用者的 `Gemini API Key`、選擇的模型名稱（預設提供 `gemini-2.5-flash` 與 `gemini-2.5-pro` 選項）及閱讀偏好設定，必須自動儲存於瀏覽器 `LocalStorage`，重新整理頁面後自動載入，無需重複輸入。

---

## 3. 核心功能與演算法規範

### 3.1 PDF 解析與文字清理邏輯
* **段落重組演算法：** 讀取 PDF 每頁文字時，必須處理學術論文常見的斷行問題。演算法需自動合併句子中間的硬換行（Hard line breaks），僅在偵測到真實段落間距或句尾標點換行時保留分段。
* **雜訊過濾與智能切分 (Chunking)：**
  * 過濾純頁碼、頁首/頁尾重複短字串等雜訊。
  * 將清理後的全文依「段落邊界」切分為每組約 **2,500 ～ 3,500 字元** 的區塊（Chunks），確保不切斷完整句子，並避免超出單次 API 輸出長度上限。

### 3.2 Google Gemini API 串接與雙階段處理
* **API 呼叫規範：** 使用標準 `fetch` 呼叫 Google Gemini REST API (`https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={apiKey}`)，並實作指數退避重試機制（Exponential Backoff）與速率限制（Rate Limit / HTTP 429）緩衝等待，避免連續請求失敗。
* **階段一：全篇結構化重點整理**
  * 提取論文關鍵章節（優先包含 Title, Abstract, Introduction, Methods, Results, Conclusion，若全文過長則智慧截取頭尾核心段落），要求 Gemini 以**繁體中文**輸出固定五段式 Markdown 導讀：
    1. **一句話核心貢獻**
    2. **研究背景與解決的痛點**
    3. **核心方法與實驗設計**
    4. **關鍵實驗數據與結果**
    5. **研究限制與未來方向**
* **階段二：逐段學術翻譯佇列**
  * 依序將切分好的區塊送交 Gemini 翻譯，每完成一段即**即時渲染至畫面並更新進度條**（支援串流式體驗或逐段非同步更新，讓使用者無需乾等整篇完成）。
  * **翻譯 Prompt 強制規則：**
    * 目標語言為**繁體中文（台灣學術用語習慣）**。
    * **專有名詞首次出現時，必須保留英文原文括號註記**，格式為：`中文譯名 (English Term, 縮寫)`，例如：`表面電漿子共振 (Surface Plasmon Resonance, SPR)`。
    * 完整保留原文的 LaTeX 數學公式（`$ ... $` / `$$ ... $$`）、Markdown 表格與程式碼區塊，嚴禁破壞公式語法。

---

## 4. UI/UX 視覺與閱讀體驗規範
1. **極簡深色護眼主題：**
   * 預設採用低疲勞深色配色，所有顏色必須透過 CSS 變數定義於 `:root`：
     * 主背景色 (`--bg-main`): `#121212`
     * 卡片與側邊欄背景 (`--bg-card`): `#1E1E1E`
     * 邊框與分隔線 (`--border-color`): `#2E2E2E`
     * 主要文字 (`--text-primary`): `#E4E4E7`
     * 次要輔助文字 (`--text-secondary`): `#A1A1AA`
     * 強調互動色 (`--accent-color`): `#3B82F6`
2. **雙欄工作區佈局：**
   * **頂部導覽與控制列：** 包含「視圖切換分頁標籤（預留未來擴充用）」、PDF 拖曳/點擊上傳區、API Key 輸入框（密碼遮罩）、模型選擇下拉選單、即時處理進度條與狀態提示。
   * **左側面板（約 35% 寬度，可獨立捲動）：** 固定顯示「論文重點結構化整理」卡片，支援一鍵複製重點內容。
   * **右側面板（約 65% 寬度，可獨立捲動）：** 為主閱讀區，頂部需提供**「純繁中閱讀」**與**「中英雙語逐段對照」**一鍵切換開關。切換至雙語模式時，每個段落需呈現「英文原文」與「對應繁體中文譯文」上下或左右對齊。

---

## 5. 內部模組化與未來可擴充規範 (Extensibility Rules)
為了確保未來在單一 `index.html` 中新增頁面（如「歷史紀錄」、「術語表」或「筆記匯出」）時不會破壞既有功能，AI 開發者**必須嚴格遵守以下結構隔離紀律**：

### 5.1 強制程式碼分區與錨點註解
`index.html` 的 `<style>`、`<body>` 與 `<script>` 必須使用以下明確的錨點註解劃分區塊，未來新增功能只能擴充對應區塊，嚴禁打散架構：
* **HTML 視圖容器規範：**
  * 所有主要頁面必須包裹在 `<section id="view-{name}" class="page-view">` 中（初始主頁面為 `id="view-translator"`）。
  * 頂部導覽列必須內建視圖切換器，透過統一的 `switchView(viewId)` 函式控制顯示與隱藏（切換 `.hidden` 類別），嚴禁使用破壞版面的彈出視窗硬塞新頁面。
* **JavaScript 四大獨立分區：**
  1. `// === [SECTION 1: STATE & CONFIG] ===`（集中管理全域狀態物件 `AppState` 與 `LocalStorage` 讀寫鍵名）
  2. `// === [SECTION 2: GEMINI API SERVICE] ===`（封裝所有對 Google Gemini API 的請求、重試與錯誤處理）
  3. `// === [SECTION 3: DOMAIN LOGIC & PARSERS] ===`（PDF 解析、文字清理、切分演算法與未來新模組的純邏輯函式）
  4. `// === [SECTION 4: UI RENDER & ROUTING] ===`（DOM 渲染、進度條更新、KaTeX/Markdown 觸發與 `switchView` 路由控制）

### 5.2 增量修改與防破壞協定 (Non-Destructive Protocol)
1. **禁止省略程式碼：** 當使用者要求修改既有 `index.html` 以新增頁面或功能時，**嚴禁使用 `// ...其餘程式碼保持不變` 等省略註解截斷檔案**，每次輸出都必須是完整、可直接運行的 `index.html`。
2. **UI 風格繼承：** 任何新增的頁面或元件，必須直接複用 `:root` 中定義的 CSS 變數（`--bg-main`、`--bg-card` 等）與既有的按鈕/卡片樣式類別，確保全站視覺一致。
3. **回歸保護清單：** 每次修改後，必須確保以下核心功能與資料結構完好無損：
   * `LocalStorage` 中的 API Key 與設定鍵名不被覆蓋。
   * PDF 上傳解析與段落合併功能正常運作。
   * 五段式重點整理與中英雙語對照切換正常運作。
4. **架構地圖同步更新：** 每次新增頁面或重大功能後，AI 需主動更新本文件底部「第 6 節：目前架構地圖」的頁面與函式清單。

---

## 6. 目前架構地圖 (Architecture Map — 供 AI 迭代參考)
* **已定義頁面視圖 (`<section class="page-view">`)：**
  * `view-translator`：主論文上傳、五段式重點整理與雙語全文翻譯工作區（預設啟用）。
* **核心狀態鍵名 (`LocalStorage`)：**
  * `paper_translator_api_key`：儲存 Google Gemini API Key。
  * `paper_translator_model`：儲存選擇的 Gemini 模型名稱。
* **核心接口函式：**
  * `switchView(viewId)`：切換不同功能頁面視圖。
  * `callGeminiAPI(systemPrompt, userContent)`：統一對外呼叫 Gemini API 的非同步函式（未來新頁面可直接複用此函式呼叫 AI）。
