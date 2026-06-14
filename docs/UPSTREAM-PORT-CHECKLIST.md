# 推回上游 VideoTogether 的檢查清單 / Upstream Port Checklist

> 目標：把這個 fork 的改動，乾淨地貢獻回原作者的 VideoTogether 專案。
> 本檔列出所有「fork 專屬、推回上游前要先處理」的項目，避免遺漏。

整個專案分兩個 git：**前端**（userscript／擴充，repo 根目錄）和**設定頁**（`source/setting/`，內嵌的獨立 repo）。兩邊都要檢查。

---

## 1. 連結與部署位置（最重要）

- [ ] **設定頁網址 `VT_SETTING_PAGE_URL`**（`source/extension/vt.js` 最上方一行）
      推回上游時應**指回上游的設定頁**（上游的 GitHub Pages），我們自己部署的設定頁只當次選／備援。
      改完記得 `python3 script/build_extension.py` 重建。
- [ ] **面板齒輪 `href`**（`source/extension/html/pannel.html` 的 `#videoTogetherSetting`）
      同上，與 `VT_SETTING_PAGE_URL` 保持一致。
- [ ] **API／更新伺服器 `source/extension/config/release_host`**
      目前已是上游的 `https://vt.panghair.com:5000/`，通常不用改；若我們曾改成自架，要還原。
- [ ] **設定頁本身**：上游有自己的設定 repo（`VideoTogether/setting`）。我們對設定頁的改動
      （繁中 zh-tw、版面重做、ⓘ 就地展開說明、新開關…）要以 PR 形式提到**上游的設定 repo**，
      而不是只留在我們 fork 的設定 repo。

## 2. 預設值差異（決定要不要一起帶上游）

- [ ] 我們把部分新安裝預設值調整過，確認上游是否接受：
  - `EnableMessageVoice` 預設 **關**（`source/extension/vt.js` getter 預設值 + 設定頁 `settingItems`）。
  - 其餘（迷你小窗、密碼房、卡頓暫停、回聲消除＝開；最小化、不自動跳轉、跳過片頭＝關）多與上游一致。
- [ ] 「關閉原生 M3U8」在設定頁被**鎖定為開**且加了說明——上游若想保留可切換，需改回。

## 3. 在地化（i18n）

- [ ] 我們在**設定頁**新增了 `zh-tw` 一整套（上游原本只有 zh-cn / en-us / ja-jp）。
- [ ] 前端在地化新增的字串（`source/extension/localization/*.json`）：`host_handed_over`、`err_*` 等，
      四語都要齊全才好被上游接受。
- [ ] 語言下拉新增「Auto（自動偵測）」選項與「空值＝自動偵測」的修正（`extension.js` + 設定頁 `getDisplayLanguage`）。

## 4. 功能性改動（這些是貢獻重點，PR 時講清楚）

- [ ] 介面／體驗重做（深色玻璃主題、淺深切換、面板 flex 版面、全螢幕迷你介面…）。
- [ ] Bug 修復：退出房間後殘留 UI、房主被接手時自動降為觀眾並跟隨新房主。
- [ ] 「預設最小化」開關失效修復（`MinimiseDefault` 優先於本站記憶）。

## 5. 不要帶進上游 PR 的 fork 專屬檔案

- [ ] `handoff/`、`docs/superpowers/`、`docs/UPSTREAM-PORT-CHECKLIST.md`（本檔）、`docs/SETTINGS-LINKAGE.md`、
      `CHANGES.md`（fork 自己的變更紀錄）、memory 等，都是 fork 內部用，PR 前排除。

## 6. 提交與品牌

- [ ] commit 訊息：上游可能偏好英文／squash 成單一乾淨提交；繁中提交訊息酌情整理。
- [ ] 全專案品牌一律 **VideoTogether**（不要出現 AniméSync 等 fork 暫用名）。
- [ ] 授權：沿用上游 MIT，不另外宣告。

---

> 小抄：要重新指向設定頁，只改 `vt.js` 的 `VT_SETTING_PAGE_URL` 與 `pannel.html` 的齒輪 `href` 兩處，重建即可。
