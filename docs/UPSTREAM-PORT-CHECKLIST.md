# 推回上游 VideoTogether 的檢查清單 / Upstream Port Checklist

> 目標：把這個 fork 的改動，乾淨地貢獻回原作者的 VideoTogether 專案。
> 本檔列出所有「fork 專屬、推回上游前要先處理」的項目，避免遺漏。
>
> **勾選約定**：`[x]` = 目前已是上游的值／已確認乾淨，推回時**不用改**；`[ ]` = fork 專屬或待辦，推回前要處理。

整個專案分兩個 git：**前端**（userscript／擴充，repo 根目錄）和**設定頁**（`source/setting/`，內嵌的獨立 repo）。兩邊都要檢查。

---

## 1. 連結與部署位置（最重要）

### 1a. 設定頁網址（目前指向我們 fork 的 lcy000，推回要還原成上游）
- [ ] **`VT_SETTING_PAGE_URL`**（`source/extension/vt.js:21`）
      目前 = `https://lcy000.github.io/VideoTogether-setting/v3.html` → 推回時改回**上游的設定頁**。改完 `python3 script/build_extension.py disable_network` 重建。
- [ ] **面板齒輪 `href`**（`source/extension/html/pannel.html:48` 的 `#videoTogetherSetting`）
      與 `VT_SETTING_PAGE_URL` 保持一致。

### 1b. 其他 lcy000 / fork 專屬引用（原本漏列，這次補上）
- [ ] **信任網域**：`source/extension/extension.js:285` 的 `'lcy000.github.io'`（以及 `:348` 的 `endsWith("lcy000.github.io")`）——這是我們 fork 設定頁的網域，推回時移除。
- [ ] **build 腳本**：`script/build_extension.py`（約 175–177 行）會 `git clone` **`LCY000/VideoTogether-setting`** 到 `source/setting`——推回時改成 clone 上游 **`VideoTogether/VideoTogether-setting`**。
- [ ] **開發文件**：`docs/en/development.md`、`docs/zh-cn/development.md` 內提到 `LCY000/VideoTogether-setting`、`lcy000.github.io`——只要把這些 **fork 專屬網址**改成上游（`VideoTogether/VideoTogether-setting`）即可。
      > `source/setting` 這個「把設定頁 clone 進來一起開發」的**安排本身是通用做法，不用排除或修正**，文件照常保留即可。

### 1c. 伺服器位址（已是上游，免改）
- [x] **API／更新伺服器 `source/extension/config/release_host`** = `https://vt.panghair.com:5000/`
      ——這就是上游 VideoTogether 的伺服器，已確認，推回**不用改**。（`debug_host` = `http://127.0.0.1:5001/` 也是上游慣用，免改。）
      > 註：`vt.js` 的 `@namespace` / `@icon` 用 `2gether.video`，那是上游官方品牌網域（非伺服器、也非 fork 專屬）；實際 API/WS 連的是上面的 `release_host`。

### 1d. 設定頁本身的改動要 PR 到上游設定 repo
- [ ] 我們對設定頁的改動（繁中 zh-tw、版面重做、ⓘ 就地展開說明、新開關…）要以 PR 形式提到上游的設定 repo **`VideoTogether/VideoTogether-setting`**，而不是只留在我們 fork 的 `LCY000/VideoTogether-setting`。

## 2. 預設值差異（決定要不要一起帶上游）

- [ ] 我們把部分新安裝預設值調整過，確認上游是否接受：
  - `EnableMessageVoice` 預設 **關**（`source/extension/vt.js` getter 預設值 + 設定頁 `settingItems`）。
  - 其餘（迷你小窗、密碼房、卡頓暫停、回聲消除＝開；最小化、不自動跳轉、跳過片頭＝關）多與上游一致。
- [ ] 「關閉原生 M3U8」在設定頁被**鎖定為開**且加了說明——上游若想保留可切換，需改回。

## 3. 在地化（i18n）

- [ ] 我們在**設定頁**新增了 `zh-tw` 一整套（上游原本只有 zh-cn / en-us / ja-jp）。
- [ ] 前端在地化新增的字串（`source/extension/localization/*.json`）：`host_handed_over`、`err_*`、`viewers_loading_hint`、`host_role`/`memeber_role`、`host_role_live`/`member_role_live`、`live_independent_hint`、`live_connected` 等，四語都要齊全才好被上游接受。
- [ ] 語言下拉新增「Auto（自動偵測）」選項與「空值＝自動偵測」的修正（`extension.js` + 設定頁 `getDisplayLanguage`）。

## 4. 功能性改動（這些是貢獻重點，PR 時講清楚）

- [ ] 介面／體驗重做（深色玻璃主題、淺深切換、面板 flex 版面、全螢幕迷你介面…）。
- [ ] Bug 修復：退出房間後殘留 UI、房主被接手時自動降為觀眾並跟隨新房主。
- [ ] 「預設最小化」開關失效修復（`MinimiseDefault` 優先於本站記憶）。
- [ ] **直播（live）觀眾各自控制**：`IsLiveStream()` 三層偵測（`duration === Infinity` ／ `live.bilibili.com` host ／ YouTube 可見 `.ytp-live-badge` ／ duration 成長啟發式）＋雙向遲滯（卡頓不閃動、誤判數秒自癒）；`SyncMemberVideo` 在直播時**不同步** seek／播放暫停／倍速，只保留「跟房主換台(URL)」。
- [ ] **直播狀態 UI**：常駐角色列「房主／觀眾 · 直播各自控制」＋第一次偵測到直播的一次性 toast＋直播時活動列由「影片同步成功」改「已連線」。
- [ ] **修一般 YouTube 影片被誤判卡在直播**（`!isFinite` 把 NaN 載入態當直播、隱藏的 `.ytp-live-badge`）。
- [ ] **popup 視窗語言偵測**對齊 `extension.js`（auto／空值不再固定變英文）。
- [ ] **工具列圖示圓角化**：`icon-16/32/48/128/192` 與執行時 `vt_64x64`/`vt_gray_64x64` 重切 24% 圓角；全螢幕聊天「送出」改低調紙飛機 icon。

## 5. 不要帶進上游 PR 的 fork 專屬檔案

- [ ] `handoff/`、`docs/superpowers/`、`docs/UPSTREAM-PORT-CHECKLIST.md`（本檔）、
      `CHANGES.md`（fork 自己的變更紀錄）、memory 等，都是 fork 內部用，PR 前排除。
- [ ] **`docs/{en,zh-cn}/development.md`** 與 **`docs/{en,zh-cn}/settings-linkage.md`** 都是通用開發說明，**可以保留／一起貢獻**；只需把 `development.md` 內的 fork 專屬網址（`LCY000`、`lcy000.github.io`）換成上游（見 1b）。`settings-linkage.md` 已用上游名稱、無 fork 網址；`source/setting` 的開發安排本身是通用做法，不必拿掉。
- [ ] **`assets/icons/`**：fork 的圖示存檔（母檔 `vt.png`、圓角圖、favicon 備份），非擴充執行所需。上游若要圓角圖示，提供 `source/chrome/icon/` 的 `icon-*` 與 `vt_64x64`/灰即可，這個存檔資料夾可不帶。

## 6. 提交與品牌

- [ ] commit 訊息：上游可能偏好英文／squash 成單一乾淨提交；繁中提交訊息酌情整理。
- [x] 全專案品牌一律 **VideoTogether**——已確認 `source/` 內**無 AniméSync 等 fork 暫用名殘留**（只剩本檔提到該名作說明）。
- [x] 授權：沿用上游 MIT，不另外宣告（無需改動）。

## 7. 版本號（PR 時交給上游決定要不要 bump、怎麼定）

版本號散在三個 manifest（各平台分開維護），userscript 則用 build 時間戳：

| 位置 | 目前 | 說明 |
| --- | --- | --- |
| `source/chrome/manifest.json` | `3.1.0` | Chrome／Edge（mv3）。本 fork 從 `3.0.23` bump 到 `3.1.0`。 |
| `source/safari/.../manifest.json` | `3.1.0` | Safari，與 chrome 同號。 |
| `source/firefox/manifest.json` | `1.4.0` | Firefox **自成一條版本線**（沿用上游 AMO 歷史，與 chrome 的 3.x 不同）。本 fork 從 `1.3.65` bump 到 `1.4.0`，**未**強行對齊成 3.x。 |
| userscript `@version`（`extension.js` 標頭）| `{{timestamp}}` | build 時自動填時間戳，非 semver，不用手動改。 |

- [ ] **要不要 bump、bump 到多少，交給上游決定**：我們負責把功能改動帶上去；版本號請原作者依他的 semver 慣例（major／minor／patch）與各商店（Chrome Web Store／AMO）的版本連續性來定。
- [ ] 若上游決定 bump：chrome／safari 通常同步一個號；firefox 是否併入同一條版本線（還是維持獨立 1.x）由上游決定。

---

> 小抄：要把設定頁／品牌指回上游，主要改這幾處再重建即可——
> `vt.js` 的 `VT_SETTING_PAGE_URL`、`pannel.html` 的齒輪 `href`、`extension.js` 的 `lcy000.github.io` 信任網域、`build_extension.py` 的設定 repo clone URL，以及兩份 `development.md` 內的 LCY000 引用。
