# AniméSync — VideoTogether 介面 / 體驗重做（變更說明）

> 本分支 `feature/anime-ui-rework` 以 `main` 為基底，是 [VideoTogether](https://github.com/VideoTogether/VideoTogether)（MIT）的 fork。
> 目標：把「和朋友一起看番」的觀看體驗做順——更現代的視覺、繁體中文、清楚的「誰在控制」狀態。
> **原則：盡量不刪上游功能**，所有改動集中在 extension 前端，方便日後維護或合併回上游。

本文件如實記錄我們實際做了什麼、技術細節、以及為什麼這樣做（體驗考量）。**未完成的部分也誠實列出，見 §9。**

---

## 改動範圍（只動這幾個檔）

| 檔案 | 改動 |
|---|---|
| `source/extension/html/pannel.html` | 主面板 markup + `<style>`，視覺重做的主場 |
| `source/extension/html/fullscreen.html` | 全螢幕迷你介面整個重做 |
| `source/extension/vt.js` | 少量接線 / 行為（主題切換、狀態文字、錯誤在地化、拖曳、全螢幕淡出…） |
| `source/extension/localization/zh-tw.json` | **新增**繁體中文（新檔） |
| `script/build_extension.py` | 註冊 `zh-tw` 為可選語言（+1 行） |

---

## 1. 在地化（繁體中文）

- **新增 `zh-tw.json`**（與 `zh-cn` 鍵值對齊），並在 `build_extension.py` 的 `languages` 註冊。
- **措辭去口語化、把「誰在控制」講白**：
  - `no_video_in_this_page`：「這個頁面沒有影片」→「尚未偵測到可同步的影片」
  - `easy_share_link_copied`：「已複製，快去分享吧」→「邀請連結已複製」
  - `host_role` →「房主（控制中）」、`memeber_role` →「觀眾（跟隨）」——直接表達誰能控制播放。

## 2. 品牌

- userscript metadata：`@name` →「AniméSync 一起看」、`@author` → AniméSync、`@namespace` → 專案 repo。
- 面板標題「VideoTogether」→「AniméSync」。
- *（誠實註記）* `@description` 已**移除原本「附彈幕」字樣**——彈幕尚未實作，見 §9。

## 3. 視覺主題（重做）

- **統一色票**（≤3 色：中性底 + 藍色 accent + 一個保留警示色），全面 token 化為 CSS 變數（`--vt-bg / text / muted / border / field / hover / accent / grad / ok / error / shadow`）。
- **深色玻璃面板**：`backdrop-filter` 霧面毛玻璃、16px 圓角、柔和景深陰影。
- **淺 / 深色模式**：
  - 預設深色，且**自動跟隨系統**（`@media (prefers-color-scheme: light)`）。
  - **可手動切換**：標題列小圓鈕 `#vtThemeToggle`，切換 `:host([data-vt-theme])`，選擇用 `localStorage("AnimeSyncTheme")` 記憶；對應 vt.js 的 `InitTheme()` / `ToggleTheme()`。
- **按鈕**：膠囊形；主鈕藍漸層 + 收斂陰影；次鈕玻璃質感；**退出鈕改中性灰（不再紅色）**——紅色容易讓人誤以為出錯。
- **狀態文字色票化**：`#videoTogetherStatusText` 依 `data-vt-status`(ok/error/info) 著色——**同步成功改藍（非綠）**、資訊灰、錯誤才警示色，且跟隨主題。
- **角色膠囊**改藍 accent（不再紅 / 粉）；**人數 icon** 從 👥 emoji 改成扁平描邊 people SVG；**愛心**從實心紅改中性描邊線條（風格統一、不搶眼）。
- 輸入框：深色玻璃、聚焦光環、placeholder 灰。

## 4. 版面 / 排版

- **面板改 flex 直欄**（header / body(flex) / footer 自然流），footer 不再 `position:absolute`——修「房內內容擁擠、footer 蓋住內容」。
- 面板高度 `auto` + `max-height: calc(100vh - 32px)`，過長則 body 內捲動。
- **房間 / 密碼列改 flex 對齊**（標籤固定寬、輸入框填滿）；在房內（input disabled）置中顯示，房名不再偏一邊；字級 / 行高對齊。
- **標題列右側按鈕** flex 行內、上下置中（修愛心 / 縮小鈕高低不齊）；縮小鈕加大提亮（20→22px）。
- `#snackbar` 改為**不佔流的浮層 toast**（原本會在 flex 欄留白）。
- **大廳收起空狀態列**（無人數 / 角色 / 狀態時以 `:has` / `:empty` 收起），避免上方大片留白。

## 5. 行為 / Bug 修正

- **沒偵測到影片不再像錯誤**：上游是 `throw Error(...)` 後紅字顯示整個 Error 物件（畫面會冒出「Error」）。改為集中在 `UpdateStatusText`：去掉 `Error:` 前綴、「還沒偵測到影片」視為一般資訊（灰、不標紅）。
- **修 Bug：退出房間後仍顯示「同步成功」**——`exitRoom()` 補上清除狀態文字 `UpdateStatusText("", "")`。
- **錯誤訊息在地化**：上游公用伺服器沒有 zh-tw、會回英文錯誤（Wrong Password / Room Not Exists / Other Host Is Syncing…）→ 在 `UpdateStatusText` 客戶端對照翻成繁中。
- **錯誤時不顯示「正在連線文字聊天伺服器…」**：收到 error 時呼叫 `setTxtMsgInterface(0)` 收起聊天介面。

## 6. 全螢幕迷你介面（`fullscreen.html`）

- **重做視覺**：黑底醜框 → 深色玻璃膠囊（blur、圓角、柔和陰影），與主面板同調。
- **移到左下角**且 `bottom: 90px`，避開最底部的播放控制條、不遮擋。
- **可自由拖動**：左側握把（people icon + 人數）按住即拖；用 `transform: translate` 相對位移，與定位脈絡無關、不會飛到角落；drag 期間才掛 `mousemove/mouseup`，放開即解除、不留監聽。
- **快速顏文字鈕**：😆😍👍🎉😂🔥 一鍵送出。
  - *（誠實註記）* 目前點擊是把該 emoji 當作**一則文字聊天訊息**送出（走上游 `SendTxtMsg` 通道），**不是彈幕**；未來才會接彈幕。
- **3 秒（後調為 2.5 秒）無動作淡出**：取代原本「✕ 永久隱藏、要重開全螢幕才回來」；滑鼠一動即重現（控制條式 UX）；打字中不淡出（input focus 暫停計時）；離開全螢幕清除監聽與計時器、無洩漏。

## 7. 多輪體驗打磨（依實測回饋）

- **修「拖移面板會變形 / 變大小」**：上游 drag 只設 `top/left`，但本 fork 把面板改 `height:auto` 後，CSS 的 `bottom/right` 與 `top/left` 並存會把面板上下左右拉伸。最終解法：主面板拖曳改用 **right/bottom（距右下角）定位**——視窗縮放時永遠貼右下角、不會飄到中間；只設 bottom 不設 top，`height:auto` 不被撐開。
- **修「切換主題 / 按下拖曳 / 按縮小時底部凸動閃爍」**：根因是面板的 `backdrop-filter` 在任何重算樣式時會重建毛玻璃圖層、造成約半秒 reflow。解法：把面板**固定為獨立合成層**（`transform: translateZ(0)` + `will-change: backdrop-filter`），毛玻璃保留、不再閃；並把深 / 淺陰影改成相同幾何（只差顏色），避免切換時底部範圍變動。
- **毛玻璃透明度 / 模糊**幾經調整，定在深 0.75 / 淺 0.70 + `blur 30px` + `saturate 170%`，讓模糊看得出來又不過白。
- **小視窗自適應**：面板寬 `min(280px, 94vw)` + `max-height: calc(100vh - 32px)`，永不超出視窗；`@media (max-width:700px)` 再縮到 240px；**視窗較小（寬<720 或高<560）時自動收成小圖示**（變大時若是自動收起的會自動展開）。
- **拖曳保留邊緣間隔**：拖到角落時與視窗邊緣保留 16px。

## 8. 刻意保留的上游功能（為利合併回上游）

- **語音通話**：曾短暫隱藏，已恢復（`voicePannel` / `callBtn` / `Voice` 程式與 UI 皆在，僅改中性配色）。
- 本機影片下載、easyshare、設定頁等上游功能皆**保留未刪**（部分標題列圖示僅在介面上精簡隱藏，功能與程式仍在）。

## 9. 尚未實作 / 已知限制（誠實揭露）

- **彈幕（螢幕飄字）尚未實作。** AniméSync 的開發倉裡有一個獨立的「彈幕分道演算法」原型（`src/danmaku/lanes.mjs`，含 6 個單元測試），但它**未整合進播放器、不會在畫面上呈現，且不包含在本分支**。畫面上目前沒有任何彈幕。
- 全螢幕的快速顏文字鈕目前是發到**文字聊天**，不是彈幕（見 §6）。
- userscript `@description` 已對應移除「附彈幕」字樣，避免誤導。

## 10. 建置 / 語言

- 以上游建置流程（`python3 script/build_extension.py` 等）即可重建 release userscript。
- `zh-tw` 已註冊為可選語言；預設體驗以繁體中文呈現。
