# 設定頁 ⇄ 前端 連動開發說明

本專案（前端 userscript / 擴充功能）與「設定頁」是兩個獨立的 git 專案，但會連動運作。
本文件說明它們怎麼連動、怎麼分開維護、以及怎麼新增「設定頁開關」。

## 兩個專案

| 專案 | 位置 | origin（我們的 fork） | upstream（原版） |
|------|------|----------------------|------------------|
| 前端 | repo 根目錄 | `LCY000/VideoTogether` | `VideoTogether/VideoTogether` |
| 設定頁 | `source/setting/`（巢狀、獨立 git） | `LCY000/VideoTogether-setting` | `VideoTogether/setting` |

`source/setting/` 是一個**獨立的 git 儲存庫**（有自己的 `.git`），在主 repo 裡是 untracked 的內嵌 repo，
所以兩邊各自 commit / push / 更新，互不干擾 = 「分開 git」。

- 從原版同步更新：`cd source/setting && git fetch upstream && git merge upstream/main`
- 推到我們的 fork：`cd source/setting && git push origin main`
- 主 repo 的建構腳本 `script/build_extension.py` 會自動 `git clone/pull` 我們的 fork
  （已從 `VideoTogether/setting` 改指向 `LCY000/VideoTogether-setting`）。
  ⚠️ 建構時會對 `source/setting` 做 `git pull`；若該資料夾有「未 commit」的本地修改，
  請先 commit，或用 `python3 script/build_extension.py disable_network` 跳過網路拉取，避免被覆蓋。

## 連動機制（storage bridge）

設定頁與前端**不直接呼叫**，而是透過共用的 storage：

```
設定頁 (任何網域，因為擴充 content script match *://*/*)
   │  使用者切換開關
   ▼  window.postMessage({ type:15, source:"VideoTogether", data:{ key, value } })   // 15 = SetStorageValue
擴充 content script → 寫入擴充 storage
   ▼  同步回所有分頁： MessageType.SyncStorageValue
window.VideoTogetherStorage[key]   ← 前端用 getVideoTogetherStorage(key, default) 讀取
```

- 設定頁讀目前值：`source/setting/v2.buildme.html` 的 `settingItems` 陣列 + 初始化迴圈會把
  `window.VideoTogetherStorage[key]` 反映到 checkbox 的勾選狀態。
- 前端讀取：`getVideoTogetherStorage('Key', 預設值)`（vt.js 開頭附近）。

## 怎麼新增一個「設定頁開關」

以本次新增的「文字訊息語音播報」(`EnableMessageVoice`) 為例：

### 設定頁端（`source/setting/`，= fork）
1. `v2.buildme.html` 的 `settingItems` 陣列加一筆 `{ key:"EnableMessageVoice", defaultVal:true }`。
2. 在開關區塊加一段 checkbox DOM：`<a id="EnableMessageVoiceLabel"></a>` + `<input id="EnableMessageVoice" type="checkbox">`
   （id 必須等於 key；label 的 id 必須是 `<key>Label`）。可選：help tooltip 用 `getLanguageText('<key>LabelHelp')`。
3. `localization/{zh-cn,zh-tw,en-us,ja-jp}.json` 各加 `EnableMessageVoiceLabel`（與 `...LabelHelp`）。
   ✅ 已補上 `localization/zh-tw.json` 並在 `v2.buildme.html` 的 `languageDicts` 註冊 + 語言選單加入「繁體中文」。
   （原版設定頁只有 zh-cn/en-us/ja-jp；本 fork 已加 zh-tw，符合本專案繁中定位。）
4. checkbox 預設未勾＝key 為 false。`defaultVal:true` 代表「未設定時視為開」。

### 前端端（`source/extension/vt.js`）
1. 加 getter：`function getEnableMessageVoice(){ return getVideoTogetherStorage('EnableMessageVoice', true); }`
2. 在要控制的地方判斷它（本例：`gotTextMsg()` 內，關閉時不播語音、不彈出「點擊啟用語音」面板，
   但仍保留文字通知與輸入框清空）。

> 預設值務必讓「未設定 = 維持原本行為」，這樣即使設定頁還沒部署，前端也不會改變現狀。

## 本次新增/處理的兩個開關

| 開關 | storage key | 預設 | 前端控制點 | 設定頁 |
|------|-------------|------|-----------|--------|
| 關閉文字訊息語音播報 | `EnableMessageVoice` | true（開） | `vt.js` `gotTextMsg()` | 本次新增 |
| 全螢幕迷你小窗 顯示與否 | `EnableMiniBar` | true（開） | `vt.js` `getEnableMiniBar()`（既有，gate 全螢幕 mini-bar） | 原版已有，本次確認 |

> 注意：`EnableMiniBar` 是「全螢幕影片時浮出的聊天迷你列」，不是面板縮小後右下角的小 logo。

## 部署狀態：已上線（2026-06-14）

- ✅ **fork 設定頁已部署到 GitHub Pages**：`https://lcy000.github.io/VideoTogether-setting/v2.html`（含兩個新開關 + 繁中）。
  - 啟用方式：`gh api -X POST repos/LCY000/VideoTogether-setting/pages`（source=main、path=/）。
  - 設定頁原始碼（`source/setting/`）已 commit `219fd5a` 並 push 到 fork main。
- ✅ **齒輪網址統一成一個常數**：`source/extension/vt.js` 最上方的 `VT_SETTING_PAGE_URL`
  （目前＝上面那個 Pages 網址）。要再換網址（例如改用自訂網域 / Vercel）**只改這一行**，重建即可。
  firstSync 擴充與 website 兩分支都用它；`pannel.html` 的靜態 href 也已同步。

### 還沒做的最後一步（你來決定何時）
- **重新安裝 / 重載新版前端 userscript**（`release/vt.*.user.js`）到瀏覽器，齒輪才會打開新的 fork 設定頁。
  （目前你瀏覽器裡跑的舊版仍指向舊網址。）
- 之後若要改設定頁網址：改 `VT_SETTING_PAGE_URL` → `python3 script/build_extension.py` → 重載。

> 前端對開關都用安全預設（維持原行為），所以即使前端還沒重載也**不會壞**，只是還沒指到新設定頁。
