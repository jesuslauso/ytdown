# YTDown 程式架構與 AI 維護指導手冊

本文件旨在為任何人工智慧（如 ChatGPT、Claude、Manus 等）提供關於 **YTDown** 專案的深入理解，確保在未來進行功能擴展、錯誤修復或程式碼重構時，能夠遵循既有的架構設計與相容性原則。

---

## 1. 專案概述

YTDown 是一個**純前端**、**無後端依賴**的 YouTube 影音下載工具。它被設計為一個單一的 HTML 檔案（`index.html`），可直接部署於 GitHub Pages 等靜態網頁代管服務上。

### 核心理念
* **零成本與高隱私**：不使用自建的後端伺服器進行影片解析或下載，所有資料處理均在客戶端瀏覽器內完成。
* **廣泛的相容性**：支援現代瀏覽器以及較舊版本的行動裝置（如 iOS 9+ 的 Safari）。
* **無縫的使用體驗**：提供自動伺服器偵測、錯誤重試機制以及直覺的使用者介面。

---

## 2. 系統架構與關鍵技術

### 2.1 依賴的外部 API：Piped
YTDown 放棄了不穩定的 Invidious 生態，全面改用 **Piped API** 來獲取 YouTube 影片的串流資訊。
* **API 端點**：主要使用 `https://api.piped.private.coffee/streams/{videoId}` 作為預設實例。
* **備用機制**：程式內建了一個 `PIPED_APIS` 陣列，包含多個 Piped 公共實例。當主伺服器解析失敗時，系統會透過 `silentRetryAll` 函數靜默嘗試其他備用伺服器。

### 2.2 純前端處理
* **無框架設計**：未使用 React、Vue 等現代前端框架，完全依賴原生 JavaScript（Vanilla JS）進行 DOM 操作，確保檔案體積小且載入迅速。
* **ES5 語法標準**：為了最大化相容性，程式碼避免使用 `const`、`let`、箭頭函數（`=>`）以及 `async/await`，全面採用 `var` 和傳統的 `function` 宣告與 Promise 鏈。

### 2.3 檔案下載與處理
* **直接下載與封裝**：利用 Fetch API 獲取影片或音訊的位元組流（Blob），並透過 `URL.createObjectURL` 觸發瀏覽器的下載行為。
* **音訊剪輯**：使用瀏覽器內建的 **Web Audio API**（`AudioContext` 與 `OfflineAudioContext`）進行音訊解碼與裁切，並將裁切後的資料封裝為 WAV 或 MP3 格式輸出。
* **MP3 編碼**：整合了輕量級的 `lamejs` 函式庫，允許在瀏覽器端將解碼後的音訊重新編碼為 MP3 格式。

---

## 3. 核心功能模組解析

### 3.1 伺服器偵測 (`detectInstance`)
系統啟動時會自動呼叫此函數。它會先嘗試從 `piped-instances.kavin.rocks` 獲取最新的實例列表，若失敗則使用內建的靜態列表。透過向各實例發送測試請求，找出回應最快且可用的伺服器，並將其設為 `currentApi`。

### 3.2 影片解析 (`analyze` 與 `silentRetryAll`)
使用者輸入網址後，系統會提取 Video ID 並向 `currentApi` 請求串流資料。
* **錯誤處理**：若解析失敗（例如遇到限流或地區限制），程式**不會**重置 `currentApi`，而是觸發 `silentRetryAll`，靜默地輪詢其他備用 API，直到成功或全部失敗為止。

### 3.3 真實檔案大小獲取 (`fetchRealSizes`)
Piped API 回傳的 `contentLength` 有時可能不準確或為空，因此程式會在解析成功後，對每個串流 URL 發送 `HEAD` 請求以獲取真實的 `Content-Length`。
* **非同步更新**：由於 `HEAD` 請求需要時間，UI 初始不會顯示大小，待請求完成後，透過 `onAllDone` 回呼函數重新呼叫 `buildAudioQList()` 或 `buildVideoQList()` 重新渲染格式列表，確保大小顯示準確。

### 3.4 預覽播放器 (`initPreview` 與 `trim` 邏輯)
解析成功後，系統會初始化一個 HTML5 `<audio>` 元素供使用者試聽。
* **進度與剪輯**：播放器綁定了 `ontimeupdate` 事件來更新自訂的進度條。使用者可以透過介面設定剪輯的「開始」與「結束」時間，下載時 `trimAudioBuffer` 會根據設定裁切音訊。

### 3.5 批量下載 (`batchStartAll` 與 `batchDownloadNext`)
* **佇列管理**：使用者可輸入多個網址，系統會建立一個 `batchQueue` 陣列來管理每個影片的狀態（pending, analyzing, ready, downloading, done, error）。
* **並發控制**：解析過程允許並發 3 個請求（`analyzeNext` 呼叫三次），而下載過程則是**嚴格序列化**進行（透過 `batchDownloadNext` 遞迴呼叫），以避免瀏覽器記憶體耗盡或觸發 API 限流。

---

## 4. AI 維護與修改指南

當 AI 助手需要對 YTDown 進行修改時，**必須**遵循以下原則：

### 4.1 語法與相容性限制
* **禁止使用 ES6+ 語法**：絕對不可使用 `let`、`const`、箭頭函數、`async/await`、模板字串（Template Literals）或展開運算子（Spread Operator）。
* **安全的 DOM 操作**：在將變數插入 HTML 字串前，必須使用內建的 `escHtml()` 函數進行跳脫，防止 XSS 攻擊。

### 4.2 API 與網路請求
* **CORS 注意事項**：所有外部 API 呼叫必須考慮 CORS 限制。由於專案部署在 HTTPS 環境（GitHub Pages），Piped API 的跨來源請求是被允許的。若要引入新的 API，必須先確認其是否支援 `Access-Control-Allow-Origin: *`。
* **優雅降級**：在新增網路請求（如 `fetch`）時，必須包含完整的 `.catch()` 錯誤處理邏輯，並確保在請求失敗時，UI 能給予適當的反饋或靜默重試，不可讓程式崩潰。

### 4.3 檔案大小與下載邏輯
* **避免重新編碼**：除非使用者明確選擇 MP3 格式，否則應盡量保留原始串流格式（如 M4A 或 Opus），因為重新編碼（如透過 lamejs）會顯著增加檔案大小且不會提升音質。
* **狀態同步**：修改下載進度或狀態時，需確保 `setProg()` 和 UI 元素的狀態同步更新。

### 4.4 測試與除錯
* 當修改了核心邏輯（如 `analyze` 或 `fetchRealSizes`），必須考慮變數的作用域與非同步執行的順序。例如，`fetchRealSizes` 更新了資料後，必須重新呼叫渲染函數，否則 DOM 不會更新。

---

## 5. 常見問題與除錯方向

* **無限重新偵測伺服器**：檢查 `analyze` 的錯誤捕捉區塊，確保在單一影片解析失敗時，不要將 `currentApi` 設為 `null`，這會導致整個系統誤判伺服器失效。
* **檔案大小顯示不更新**：檢查 `fetchRealSizes` 是否正確執行，以及其回呼函數 `onAllDone` 是否有正確觸發 `buildAudioQList()` 重新渲染 DOM。
* **iOS 舊裝置無法播放或下載**：檢查是否誤用了 `Array.prototype.findIndex` 或 `Object.assign` 等不被舊版 Safari 支援的方法，必要時請自行實作 Polyfill。

---
*本文檔由 Manus AI 生成，專為 YTDown 專案的持續整合與維護設計。*
