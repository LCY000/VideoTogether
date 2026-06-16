# 浮動面板收合/展開狀態 重構 — 實作計畫

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 消除浮動面板「先展開再收合」的閃爍，並以「是否在房間」決定收/展：不在房間=純看設定、在房間=繼承自己上一頁的狀態（無記憶則展開）。

**Architecture:** 把「收/展決策」抽成一個純函式（可單元測試），決策來源改為（a）`MinimiseDefault` 的 localStorage 鏡像，讓 `Init()` 能**同步**決定、不必等非同步 sync；（b）跟著房間會話走的 `VideoTogetherMinimized`（存進每分頁的 TabStorage + sessionStorage，**不放 URL**，避免房主狀態傳染觀眾）。核心原則：在「確定該展開」之前絕不先展開。

**Tech Stack:** 原生 JS（content script，IIFE 包覆，無模組系統）；建置 `script/build_extension.py`（文字替換 `{{{ }}}`/`{$ $}`）；測試用 Node 內建 `assert`（無新依賴）。

**設計依據：** `docs/superpowers/specs/2026-06-17-floating-panel-collapse-state-design.md`

---

## 檔案結構

| 檔案 | 動作 | 責任 |
|---|---|---|
| `source/extension/vt.js` | 修改 | 全部邏輯：純函式 + 接線（Init / RecoveryState / InRoom / GetRoomState / SaveState / SaveIsMinimized / firstSync / exitRoom） |
| `test/extension/collapse-state.test.js` | 新增 | 抽出 vt.js 內純函式並驗證真值表（Node assert） |
| `release/*` | 重建 | 建置產物（git 追蹤），最後一次 build 後 commit |

> 純函式以 `// === collapse-state:start/end ===` sentinel 包覆，留在 vt.js 內（讓 Init 等同作用域直接呼叫），測試讀檔切片 `eval` 來測**真正的原始碼**，不複製、不改 build/manifest。

---

## Task 1：純決策函式 + 單元測試（TDD）

**Files:**
- Test: `test/extension/collapse-state.test.js`（新增）
- Modify: `source/extension/vt.js`（在 `show()` 之後，約 `:335`）

- [ ] **Step 1：寫失敗測試**

建立 `test/extension/collapse-state.test.js`：

```js
// 單元測試：抽出 vt.js 內的純函式 VideoTogetherResolveMinimized 並驗證真值表。
// 無框架，用 Node 內建 assert。執行：node test/extension/collapse-state.test.js
const fs = require('fs');
const path = require('path');
const assert = require('assert');

const vtPath = path.join(__dirname, '..', '..', 'source', 'extension', 'vt.js');
const src = fs.readFileSync(vtPath, 'utf8');
const START = '// === collapse-state:start';
const END = '// === collapse-state:end';
const s = src.indexOf(START);
const e = src.indexOf(END);
assert.ok(s !== -1 && e !== -1 && e > s,
    'sentinel 區塊未找到：請確認 vt.js 內有 collapse-state:start/end');
// eslint-disable-next-line no-eval
eval(src.slice(s, e)); // 定義 VideoTogetherResolveMinimized 於本作用域

function expect(state, want, msg) {
    assert.strictEqual(VideoTogetherResolveMinimized(state), want, msg);
}

// 在房間：繼承 carried，無則展開
expect({ inRoom: true, carried: 1 }, true, '在房間 carried=1 → 收合');
expect({ inRoom: true, carried: "1" }, true, '在房間 carried="1" → 收合');
expect({ inRoom: true, carried: 0 }, false, '在房間 carried=0 → 展開');
expect({ inRoom: true, carried: "0" }, false, '在房間 carried="0" → 展開');
expect({ inRoom: true, carried: null }, false, '在房間 無carried → 展開（觀眾首次進房）');
expect({ inRoom: true, carried: undefined }, false, '在房間 carried undefined → 展開');
// 不在房間：純看設定，未知收合（安全）
expect({ inRoom: false, minimiseDefault: true }, true, '不在房間 設定開 → 收合');
expect({ inRoom: false, minimiseDefault: false }, false, '不在房間 設定關 → 展開');
expect({ inRoom: false, minimiseDefault: null }, true, '不在房間 設定未知 → 收合（安全）');
expect({ inRoom: false }, true, '不在房間 無設定資訊 → 收合（安全）');

console.log('collapse-state: 全部通過');
```

- [ ] **Step 2：執行測試確認失敗**

Run: `node test/extension/collapse-state.test.js`
Expected: 失敗 `AssertionError: sentinel 區塊未找到...`（函式尚未加入 vt.js）

- [ ] **Step 3：在 vt.js 加入 sentinel 包覆的純函式**

在 `source/extension/vt.js` 的 `show()` 函式之後插入。用此 `old_string` 定位（`:333-335`）：

```js
    function show(e) {
        if (e) e.style.display = null;
    }
```

替換為：

```js
    function show(e) {
        if (e) e.style.display = null;
    }

    // === collapse-state:start — 純函式，單元測試見 test/extension/collapse-state.test.js ===
    // 決定面板初始 minimized：
    //   在房間   → 繼承 carried（1/"1"/true=收合、0/"0"/false=展開、缺失=展開）
    //   不在房間 → 看 MinimiseDefault（true=收合、false=展開、未知=收合「安全，絕不先展開」）
    function VideoTogetherResolveMinimized(state) {
        if (state && state.inRoom) {
            var c = state.carried;
            if (c === 1 || c === "1" || c === true) return true;
            if (c === 0 || c === "0" || c === false) return false;
            return false; // 在房間、無記憶 → 展開
        }
        var d = state ? state.minimiseDefault : undefined;
        if (d === true) return true;
        if (d === false) return false;
        return true; // 不在房間、設定未知 → 收合（安全，絕不先展開）
    }
    // === collapse-state:end ===
```

- [ ] **Step 4：執行測試確認通過**

Run: `node test/extension/collapse-state.test.js`
Expected: `collapse-state: 全部通過`（exit 0）

- [ ] **Step 5：Commit**

```bash
git add test/extension/collapse-state.test.js source/extension/vt.js
git commit -m "feat(panel): 新增收/展決策純函式 VideoTogetherResolveMinimized + 單元測試

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 2：讀取/套用路徑（Init / RecoveryState / InRoom）

讓初始與還原時都用純函式決策，且 `InRoom()` 不再強制展開。

**Files:**
- Modify: `source/extension/vt.js`（`Init()` `:1864-1871`、`InRoom()` `:1873-1877`、`RecoveryStateFrom` `:3282-3284`）

- [ ] **Step 1：改寫 `Init()`**

`old_string`（`:1864-1871`）：

```js
        Init() {
            let VideoTogetherMinimizedHere = localStorage.getItem("VideoTogetherMinimizedHere");
            if (VideoTogetherMinimizedHere == 0) {
                this.Maximize(true);
            } else if (VideoTogetherMinimizedHere == 1) {
                this.Minimize(true);
            }
        }
```

`new_string`：

```js
        Init() {
            // 同步決定初始收/展，避免等非同步 sync 才收合造成「展→收」閃爍。
            // 在房間（此時只能從 sessionStorage 同步得知；TabStorage 房間留待 firstSync 還原）→ 繼承 carried；
            // 不在房間 → 看 MinimiseDefault 的 localStorage 鏡像（未知則收合，安全）。
            let inRoom = false, carried = null;
            try {
                let ts = parseFloat(window.sessionStorage.getItem("VideoTogetherTimestamp"));
                let rn = window.sessionStorage.getItem("VideoTogetherRoomName");
                if (rn && !isNaN(ts) && ts + 60 >= Date.now() / 1000) {
                    inRoom = true;
                    carried = window.sessionStorage.getItem("VideoTogetherMinimized");
                }
            } catch (e) { }
            let minimiseDefault = null;
            try {
                let m = window.localStorage.getItem("VideoTogetherMinimiseDefault");
                if (m === "1") minimiseDefault = true;
                else if (m === "0") minimiseDefault = false;
            } catch (e) { }
            if (VideoTogetherResolveMinimized({ inRoom: inRoom, carried: carried, minimiseDefault: minimiseDefault })) {
                this.Minimize(true);
            } else {
                this.Maximize(true);
            }
        }
```

- [ ] **Step 2：Init 後補跑 autoCollapse（視窗過小才收，且參考已就緒）**

`old_string`（`:1679-1680`）：

```js
                this.InLobby(true);
                this.Init();
```

`new_string`：

```js
                this.InLobby(true);
                this.Init();
                // Init 已用參考定案初始收/展後，再跑一次 autoCollapse：視窗過小時收成圖示（此時 videoTogether(FlyPannel|SamllIcon) 參考已就緒，不再對 undefined 操作）。
                autoCollapse();
```

- [ ] **Step 3：`InRoom()` 移除強制展開**

`old_string`（`:1873-1877`）：

```js
        InRoom() {
            try {
                speechSynthesis.getVoices();
            } catch { };
            this.Maximize();
```

`new_string`：

```js
        InRoom() {
            try {
                speechSynthesis.getVoices();
            } catch { };
            // 收/展不再由 InRoom 決定：避免房主狀態傳染觀眾、避免每次還原都強制展開。
            // 改由 Init / RecoveryState / firstSync 依「是否在房間 + carried/設定」決定。
```

- [ ] **Step 4：`RecoveryStateFrom` 還原時套用 carried**

`old_string`（`:3282-3284`）：

```js
                        window.videoTogetherFlyPannel.inputRoomName.value = vtRoomName;
                        window.videoTogetherFlyPannel.inputRoomPassword.value = password;
                        window.videoTogetherFlyPannel.InRoom();
```

`new_string`：

```js
                        window.videoTogetherFlyPannel.inputRoomName.value = vtRoomName;
                        window.videoTogetherFlyPannel.inputRoomPassword.value = password;
                        window.videoTogetherFlyPannel.InRoom();
                        // 還原房間時套用 carried 收/展（缺失 → 展開）。getFunc 對應 TabStorage / sessionStorage / URL。
                        if (VideoTogetherResolveMinimized({ inRoom: true, carried: getFunc("VideoTogetherMinimized") })) {
                            window.videoTogetherFlyPannel.Minimize(true);
                        } else {
                            window.videoTogetherFlyPannel.Maximize(true);
                        }
```

- [ ] **Step 5：建置 + 語法檢查**

Run:
```bash
python3 script/build_extension.py disable_network 2>&1 | tail -3
node --check release/vt.user.js && echo "SYNTAX OK"
```
Expected: build 完成、`SYNTAX OK`（無 SyntaxError）

- [ ] **Step 6：Commit（只 commit source，release 留到最後一次建置）**

```bash
git add source/extension/vt.js
git commit -m "feat(panel): 讀取路徑改用決策函式；InRoom 不再強制展開

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 3：持久化/權威路徑（GetRoomState / SaveState / SaveIsMinimized / firstSync / exitRoom）

讓 `VideoTogetherMinimized` 跟著房間持久化、firstSync 寫鏡像並只在「不在房間」時套設定、退房清記憶。

**Files:**
- Modify: `source/extension/vt.js`（`SaveIsMinimized` `:1839-1841`、firstSync `:2976-2986`、exitRoom `:3359-3362`、`GetRoomState` `:3760-3769`、`SaveStateToSessionStorageWhenSameOrigin` `:3784-3790`）

- [ ] **Step 1：`GetRoomState()` 新增 `VideoTogetherMinimized`**

`old_string`（`:3760-3769`）：

```js
            return {
                VideoTogetherUrl: link,
                VideoTogetherRoomName: this.roomName,
                VideoTogetherPassword: this.password,
                VideoTogetherRole: this.role,
                VideoTogetherTimestamp: Date.now() / 1000,
                VideoTogetherVoice: voice,
                VideoVolume: this.getVideoVolume(),
                VoiceVolume: this.getVoiceVolume()
            }
```

`new_string`：

```js
            return {
                VideoTogetherUrl: link,
                VideoTogetherRoomName: this.roomName,
                VideoTogetherPassword: this.password,
                VideoTogetherRole: this.role,
                VideoTogetherTimestamp: Date.now() / 1000,
                VideoTogetherVoice: voice,
                VideoVolume: this.getVideoVolume(),
                VoiceVolume: this.getVoiceVolume(),
                // 收/展跟著房間會話跨頁繼承（每個用戶端各自的；刻意不放 URL，避免傳染給觀眾）
                VideoTogetherMinimized: (window.videoTogetherFlyPannel && window.videoTogetherFlyPannel.minimized) ? 1 : 0
            }
```

- [ ] **Step 2：`SaveStateToSessionStorageWhenSameOrigin()` 寫入 minimized**

`old_string`（`:3784-3790`）：

```js
                if (link == "" || sameOrigin) {
                    window.sessionStorage.setItem("VideoTogetherUrl", link);
                    window.sessionStorage.setItem("VideoTogetherRoomName", this.roomName);
                    window.sessionStorage.setItem("VideoTogetherPassword", this.password);
                    window.sessionStorage.setItem("VideoTogetherRole", this.role);
                    window.sessionStorage.setItem("VideoTogetherTimestamp", Date.now() / 1000);
                    return sameOrigin;
                }
```

`new_string`：

```js
                if (link == "" || sameOrigin) {
                    window.sessionStorage.setItem("VideoTogetherUrl", link);
                    window.sessionStorage.setItem("VideoTogetherRoomName", this.roomName);
                    window.sessionStorage.setItem("VideoTogetherPassword", this.password);
                    window.sessionStorage.setItem("VideoTogetherRole", this.role);
                    window.sessionStorage.setItem("VideoTogetherTimestamp", Date.now() / 1000);
                    window.sessionStorage.setItem("VideoTogetherMinimized",
                        (window.videoTogetherFlyPannel && window.videoTogetherFlyPannel.minimized) ? 1 : 0);
                    return sameOrigin;
                }
```

- [ ] **Step 3：改寫 `SaveIsMinimized()`（手動收/展只在房間持久化）**

`old_string`（`:1839-1841`）：

```js
        SaveIsMinimized(minimized) {
            localStorage.setItem("VideoTogetherMinimizedHere", minimized ? 1 : 0)
        }
```

`new_string`：

```js
        SaveIsMinimized(minimized) {
            // 收/展只在「房間會話」中記憶（跟著房間跨頁繼承）；不在房間則不記憶（沒在房間 → 純看設定）。
            // 立即寫入 TabStorage + sessionStorage，避免手動操作後馬上換頁、來不及被同步迴圈持久化。
            // 註：this.minimized 已由 Minimize/Maximize 先行設定，GetRoomState 讀的就是它。
            try {
                let ext = window.videoTogetherExtension;
                if (ext && ext.role != ext.RoleEnum.Null) {
                    let state = ext.GetRoomState("");
                    sendMessageToTop(MessageType.SetTabStorage, state);
                    ext.SaveStateToSessionStorageWhenSameOrigin("");
                }
            } catch (e) { }
        }
```

- [ ] **Step 4：改寫 firstSync 決策 + 寫鏡像**

`old_string`（`:2976-2986`）：

```js
                    if (firstSync) {
                        // 全域「預設最小化」(MinimiseDefault) 優先：開啟時每次載入都先收成右下角小圖示，
                        // 即使本站之前手動展開/收合過也一樣（Init() 讀 VideoTogetherMinimizedHere 會把 disableDefaultSize 設成 true，
                        // 舊版寫法會因此整段被跳過，導致此開關「看起來完全沒作用」）。
                        // 關閉時才尊重本站記憶：已有 disableDefaultSize（Init 已套本站狀態）就不動，否則預設展開。
                        if (data.MinimiseDefault) {
                            window.videoTogetherFlyPannel.Minimize(true);
                        } else if (!window.videoTogetherFlyPannel.disableDefaultSize) {
                            window.videoTogetherFlyPannel.Maximize(true);
                        }
                    }
```

`new_string`：

```js
                    if (firstSync) {
                        // 把 MinimiseDefault 鏡像進 localStorage，供下次載入 Init() 同步讀取（消除「展→收」閃爍的關鍵）。
                        try { localStorage.setItem("VideoTogetherMinimiseDefault", data.MinimiseDefault ? 1 : 0); } catch (e) { }
                        // 權威決策：不在房間 → 純看設定；在房間 → 已由上方 RecoveryState 依 carried 套好，這裡不覆寫。
                        // （this.role 在 RecoveryState 後即反映是否在房間。）
                        if (this.role == this.RoleEnum.Null) {
                            if (data.MinimiseDefault) {
                                window.videoTogetherFlyPannel.Minimize(true);
                            } else {
                                window.videoTogetherFlyPannel.Maximize(true);
                            }
                        }
                    }
```

- [ ] **Step 5：`exitRoom()` 清掉 sessionStorage 的 minimized**

`old_string`（`:3359-3362`）：

```js
            window.videoTogetherFlyPannel.InLobby();
            let state = this.GetRoomState("");
            sendMessageToTop(MessageType.SetTabStorage, state);
            this.SaveStateToSessionStorageWhenSameOrigin("");
```

`new_string`：

```js
            window.videoTogetherFlyPannel.InLobby();
            let state = this.GetRoomState("");
            sendMessageToTop(MessageType.SetTabStorage, state);
            this.SaveStateToSessionStorageWhenSameOrigin("");
            // 退房清掉房間會話的收/展記憶；之後回到「不在房間 → 純看設定」。
            // TabStorage 因 role=Null 時 GetRoomState 回傳 {} 已被清空。
            try { window.sessionStorage.removeItem("VideoTogetherMinimized"); } catch (e) { }
```

- [ ] **Step 6：建置 + 語法檢查**

Run:
```bash
python3 script/build_extension.py disable_network 2>&1 | tail -3
node --check release/vt.user.js && node --check release/vt.website.js && echo "SYNTAX OK"
```
Expected: build 完成、`SYNTAX OK`

- [ ] **Step 7：跑單元測試確保純函式仍正確（回歸）**

Run: `node test/extension/collapse-state.test.js`
Expected: `collapse-state: 全部通過`

- [ ] **Step 8：Commit（只 source）**

```bash
git add source/extension/vt.js
git commit -m "feat(panel): minimized 跟房間持久化；firstSync 寫鏡像並只在非房間套設定；退房清記憶；淘汰每網域 VideoTogetherMinimizedHere

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 4：建置產物 + 行為驗證 + 收尾

**Files:**
- Modify: `release/*`（重建後 commit）
- 參考：`CHANGES.md`（依專案慣例補一行變更紀錄，可選）

- [ ] **Step 1：乾淨重建全部 release**

Run:
```bash
python3 script/build_extension.py disable_network 2>&1 | tail -5
git status --short release | head
```
Expected: release 下相關建置檔更新（vt.*.user.js / vt.*.website.js 等）

- [ ] **Step 2：行為驗證（在真實動畫站手動測，逐項打勾）**

> 自動化測不到「不閃」的時序（屬 content script 載入時序），以真實站點手動驗證為準；如需可另用 `test/extension/sync.js`（puppeteer）跑房間流程。每項先確保該網域 localStorage 已有 `VideoTogetherMinimiseDefault` 鏡像（即「載入過一次後」）。

- [ ] 設定「開」、不在房間、刷新 → 收合，**全程無展開閃爍**
- [ ] 設定「關」、不在房間、刷新 → 展開（該網域首次允許一次「收→展」）
- [ ] 在房間、上頁展開、換下一集 → 展開（繼承）
- [ ] 在房間、上頁收合、換下一集 → 收合（繼承）
- [ ] 觀眾點邀請連結首次進房 → 展開
- [ ] 房主把面板收合後換頁，另一台觀眾不受影響（不被連帶收合）
- [ ] 退房後刷新 → 行為回到「純看設定」，無殘留
- [ ] 縮小視窗 → 仍自動收合（autoCollapse 未壞）

- [ ] **Step 3：（可選）CHANGES.md 補一行**

依專案慣例在 `CHANGES.md` 最新版本區塊加入：
```
- 修正浮動面板載入時「先展開再收合」的閃爍；收/展改以「是否在房間」決定（不在房間=依設定、在房間=繼承上一頁，觀眾首次進房展開），退房清除記憶。
```

- [ ] **Step 4：Commit 建置產物（與可選文件）**

```bash
git add release CHANGES.md
git commit -m "build: 重建 release（浮動面板收/展行為重構）

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## 備註 / 已知取捨

- **dead code**：`disableDefaultSize`（`:1824`、`:1834` 賦值）在改寫 firstSync 後不再被讀取；保留無害，未在本計畫移除以縮小改動面。
- **設定 OFF 首訪某網域**：鏡像尚未建立 → Init 先收合、firstSync 後展開（一次「收→展」）。之後該網域同步正確、不再閃。符合 spec「務實：只消除展→收」。
- **跨網域且只走 URL handoff**（無 TabStorage 亦無同源 sessionStorage）：minimized 不繼承 → 回到「在房間無 carried → 展開」。符合 spec 邊界。
- **設定剛從 關→開、同網域尚未再 sync**：該網域下一次載入可能出現一次「展→收」；任一頁 sync 後鏡像即更新、之後消失。視窗極小，可接受。
