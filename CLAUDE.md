# CLAUDE.md — GIF Maker

專案說明，給 Claude Code 讀。請遵守以下原則編輯。

## 這是什麼
純前端的「影片轉 GIF」網頁工具，跑在使用者瀏覽器裡（用 ffmpeg.wasm），
不需後端、不上傳檔案。部署目標是 GitHub Pages。

## 重要架構約束（不要破壞）
- **單一檔案**：所有 HTML / CSS / JS 都在 `index.html`。維持單檔，方便部署與分享。
- **單執行緒 ffmpeg.wasm**：刻意不用多執行緒版，因為 GitHub Pages 無法設定
  COOP/COEP headers（SharedArrayBuffer 需要）。**不要**改成需要 cross-origin
  isolation 的版本，否則 GitHub Pages 會壞掉。
- **無後端、無 build step**：不要引入 npm build、bundler、framework。
  直接用 `<script type="module">` + ESM CDN import。
- 依賴透過 CDN 載入：`@ffmpeg/ffmpeg`、`@ffmpeg/util` 走 esm.sh；
  **`@ffmpeg/core` 與 worker 走 unpkg**（原因見下方「ffmpeg.wasm 載入配方」，
  別擅自統一改回 esm.sh，會壞）。

## ⚠️ ffmpeg.wasm 載入配方（極度反直覺，動之前先讀完）
這段在 `ensureFFmpeg()`。曾經連環踩四個雷才調好，每個改動都會整個壞掉，
**不要為了「統一」或「看起來乾淨」而改**。規則如下：

1. **必須透過 http server 開**（`file://` 雙擊永遠不會動）。
   跨來源的 module / worker / wasm 在 `file://` 下會被擋且常無聲卡死。
   `convert()` 開頭已有 `location.protocol==="file:"` 攔截並提示，別拿掉。

2. **Worker 必須同源**：ffmpeg.wasm 內部 `new Worker()`，瀏覽器規定 Worker
   script 必須同源，直接指 CDN 會被擋（`file://` 報 origin 'null'，http 報
   SecurityError）。解法：建一個同源 blob worker 去 `import` CDN 上的真 worker：
   ```js
   const classWorkerURL = URL.createObjectURL(
     new Blob([`import "${FFM}/worker.js";`], { type: "text/javascript" }));
   // FFM = "https://unpkg.com/@ffmpeg/ffmpeg@0.12.10/dist/esm"
   ```
   傳給 `ffmpeg.load({ classWorkerURL, ... })`。blob 繼承頁面來源 → 同源；
   blob 內用絕對 URL import，真 worker 的相對依賴(`./const.js`)會解析回 CDN。

3. **core 必須用 unpkg 的「原始 ESM」檔，且 coreURL 不可 blob 化**：
   - 不可用 esm.sh 的 core：esm.sh 會把 node 模組改寫成 unenv polyfill，
     瀏覽器跑起來丟 `module.require is not implemented`。
   - 不可 `toBlobURL` core：worker 是 module 型，用 `import()` 載 core；
     core 內有 `import.meta.url` 與根路徑 import，base 變成 `blob:` 會炸
     （`Failed to resolve module specifier "/node/buffer.mjs"`）。
   - 不可用 UMD core：module worker 沒有 `importScripts`，且 UMD 無 default export。
   - 正解：`coreURL: \`${CORE}/ffmpeg-core.js\``（CORE = unpkg dist/esm，直傳網址）。

4. **wasm 可以、且應該 blob 化**：純 binary、無 import，
   `wasmURL: await toBlobURL(\`${CORE}/ffmpeg-core.wasm\`, "application/wasm")`。

5. **錯誤要看得到真因**：有掛 `ffmpeg.on("log")` 存進 `ffmpegLog`，catch 會把
   真實錯誤 + log 尾巴顯示出來。別改回那句通用的 "try lower fps"。

一句話記法：**worker→同源blob、core→unpkg原始ESM直傳、wasm→blob，缺一不可。**

## 編輯習慣（重要，省 token）
- 優先用 `str_replace` 做局部編輯，**不要**整檔重寫。
- 修改前先 `view` 相關區段，不要盲改。
- CSS 變數集中在 `:root`，改顏色/間距從那裡動。

## 設計規格（已確認，勿擅自更動）
- Accent 紅色：`#E63E25`
- 字體：Sora（UI）+ DM Mono（數字/檔名）
- 操作文字用英文、說明文字可中文（混合）
- 四步驟流程：上傳 → 來源資訊+設定 → 轉換進度 → 結果比較

## 功能清單（目前狀態 = 全部完成）
1. 拖拉 / 點擊選檔，200 MB 上限
2. 讀取後顯示第一 frame、檔案大小、dimension、時長
3. 設定：FPS（流暢度）、Speed（setpts 縮短時間）、Size（預設選項，鎖長寬比）
4. 轉換真實進度條（ffmpeg progress callback）
5. 結果 before/after 比較大小與尺寸
6. 網頁內預覽 GIF
7. 下載：File System Access API（Chrome/Edge 選資料夾）+ 標準下載 fallback

## ffmpeg 轉換邏輯（filter chain）
```
setpts={1/speed}*PTS, fps={fps}, scale={width}:-1:flags=lanczos,
split[s0][s1];[s0]palettegen[p];[s1][p]paletteuse
```
- `setpts` 控速、`fps` 控流暢度，兩者獨立。
- 用 palettegen/paletteuse 兩段式產生較高畫質 GIF。

## 已知限制（不是 bug，別「修」）
- GIF 常比原影片大，這是 GIF 格式特性，介面已誠實標示。
- 很大或很長的影片可能因瀏覽器記憶體不足而失敗，已有 try/catch 提示。
- 第一次載入需下載 ~25MB 核心，之後瀏覽器快取。

## 本地預覽
```bash
python3 -m http.server 8000
# 開 http://localhost:8000
```
注意：用 file:// 直接開可能因 module CORS 失敗，請用 http server。

## 可能的未來工作（user 想做時再做）
- 裁切時間範圍（trim start/end）
- 自訂寬度輸入框（目前是預設選項）
- 拖曳調整輸出區域
