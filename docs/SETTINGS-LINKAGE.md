# 前端 ⇄ 設定頁 連動與開發 / Frontend ⇄ Settings-page linkage

> 本文說明「前端（userscript／擴充）」與「設定頁」這兩個獨立專案**怎麼連動、怎麼分開維護、怎麼新增一個設定開關**。
> 環境準備、編譯、載入擴充等請見 [development.md](zh-cn/development.md)（[English](en/development.md)）。

## 1. 兩個專案

| 專案 | 位置 | 上游（原版） |
|------|------|------|
| 前端（userscript／擴充） | repo 根目錄 | `VideoTogether/VideoTogether` |
| 設定頁 | `source/setting/`（嵌套的獨立 git） | `VideoTogether/VideoTogether-setting` |

`source/setting/` 是一個**嵌套的獨立 git checkout**（有自己的 `.git`，被主 repo `.gitignore` 忽略、非 submodule）。
兩邊各自 commit / push / 同步上游，互不干擾。建構腳本 `script/build_extension.py` 會把它 `clone` 進來一起建構。
（這個安排與如何修改設定頁，詳見 [development.md 的 `source/setting` 段](zh-cn/development.md#source-setting-与编译时会拉取的其他仓库)。）

## 2. 連動機制（storage bridge）

設定頁與前端**不直接互相呼叫**，而是透過擴充共用的 storage 溝通：

```
設定頁（任何網域皆可，因為 content script match *://*/*）
   │  使用者切換開關
   ▼  window.postMessage({ source:"VideoTogether", type:15, data:{ key, value } })   // 15 = SetStorageValue
擴充 content script  →  寫入擴充 storage
   ▼  廣播回所有分頁（MessageType.SyncStorageValue）
window.VideoTogetherStorage[key]   ←  前端用 getVideoTogetherStorage(key, default) 讀取
```

- **設定頁讀目前值**：設定頁的 `settingItems` 陣列 + 初始化迴圈，會把 `window.VideoTogetherStorage[key]` 反映到 checkbox 勾選狀態。
- **前端讀值**：`getVideoTogetherStorage('<Key>', <預設值>)`（`source/extension/vt.js` 開頭附近）。
- **信任網域**：設定頁網域要在 `source/extension/extension.js` 的信任清單裡（`2gether.video`、`*.github.io` 等），讀寫 storage 才被允許。開發時把 `isDevelopment` 設 `true` 可讓任意網域都受信任（見 development.md）。

## 3. 怎麼新增一個「設定開關」

以一個假想開關 `EnableFoo`（預設開）為例。**兩邊都要改**：

### 設定頁端（`source/setting/`）
1. 在 `settingItems` 陣列加一筆：`{ key:"EnableFoo", defaultVal:true }`。
2. 在開關區塊加 checkbox DOM：`<a id="EnableFooLabel"></a>` + `<input id="EnableFoo" type="checkbox">`
   （**input 的 id 必須等於 key；label 的 id 必須是 `<key>Label`**）。可選 help tooltip：`getLanguageText('EnableFooLabelHelp')`。
3. 各語言 `localization/*.json` 加 `EnableFooLabel`（與可選的 `EnableFooLabelHelp`），並在語言字典註冊。

### 前端端（`source/extension/vt.js`）
1. 加 getter：`function getEnableFoo(){ return getVideoTogetherStorage('EnableFoo', true); }`
2. 在要控制的地方判斷它。

> **預設值守則**：務必讓「未設定 = 維持原本行為」。這樣即使設定頁還沒部署、或前端還沒讀到新值，行為都不會改變、不會壞。

## 4. 設定頁網址（齒輪連結）

前端面板齒輪打開的設定頁網址，集中在一個常數：`source/extension/vt.js` 最上方的 `VT_SETTING_PAGE_URL`
（`source/extension/html/pannel.html` 的齒輪 `href` 與它保持一致）。要換設定頁部署位置，改這一處 → 重新編譯即可。
