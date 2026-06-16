# 浮動面板收合/展開狀態 — 重新設計

日期：2026-06-17
分支：feature/ui-rework
範圍：`source/extension/vt.js`、`source/extension/html/pannel.html`

---

## 1. 背景與問題

浮動面板的「收合（右下角小圖示）／展開」狀態，目前由四個地方在不同時間點各自決定，互相打架：

| # | 位置 | 行為 |
|---|---|---|
| ① HTML 初始 | `pannel.html:2` | `#videoTogetherFlyPannel` 內聯 `display:none`、小圖示可見 → 預設其實是**收合** |
| ② `Init()`（同步，建面板時） | `vt.js:1864-1871` | 讀 `localStorage["VideoTogetherMinimizedHere"]`（**每網域記憶**）：`0`→展開、`1`→收合、無→不動 |
| ③ `RecoveryState()→InRoom()` | `vt.js:3284`、`1877` | 還原房間時無條件 `Maximize()` → **強制展開**，且 `isDefault=false` 會把 `VideoTogetherMinimizedHere` 寫成 `0` |
| ④ `SyncStorageValue` firstSync（**非同步**） | `vt.js:2976-2985` | `MinimiseDefault` 開→`Minimize`；關→`Maximize` |

### 根因

「預設收合」設定 `MinimiseDefault` 只能透過 ④ 這個**非同步**背景訊息取得，而它到達前，②③ 這些**同步**路徑已先把面板展開。順序變成：

> 同步 Init/InRoom 先展開 →（畫面已畫出展開面板）→ 非同步 sync 才收合

時間差不固定，於是出現「先展開、再收起、閃幾下」。

**最主要的觸發源**：`VideoTogetherMinimizedHere` 是**每網域 localStorage**。使用者只要在某站進過一次房（`InRoom→Maximize()` 寫入 `0`），之後**每次重新整理**，② 都讀到 `0` 而先展開 → ④ 再收合，即使當下沒在房間也天天閃。這個「每網域記憶」是要被淘汰的設計。

---

## 2. 目標行為

收合/展開的決策，依「是否在房間」分兩種模型。

> **重要：有兩個不同的「預設」，各管各的情況、互不衝突。**
> - **設定開關的預設**（開→收合 / 關→展開）：**只在「不在房間」時生效**。
> - **「在房間但無自己的記憶」的預設**：**永遠展開**，與設定開關無關。
>
> 因此「預設收合」只影響「不在房間」的情況。觀眾點連結 / 首次進房的人，因為**已經在房間**，套的是後者 → 一律展開。

兩種模型：

### A. 不在房間
行為完全等同設定開關，**忽略任何手動記憶**：
- `MinimiseDefault` 開 → 收合
- `MinimiseDefault` 關 → 展開

每頁獨立、可預期。（使用者若覺得手動收合不被記住很煩，正解是打開設定開關——那正是該開關的用途。）

### B. 在房間
繼承「自己這條房間會話」上一頁的收合/展開狀態：
- 有 carried 狀態 → 沿用（上頁收就收、上頁展就展）
- **無 carried 狀態 → 展開**（觀眾透過邀請連結首次進房、房主首次建房，都屬此類 → 一律展開，確保看得到面板可操作）

### 關鍵子情境
- **觀眾首次進房（邀請連結）**：走 URL 參數還原，URL **不攜帶** minimized → carried 缺失 → 展開。
- **加入/建立房間當下**：面板本來就是展開的（要操作才能按加入），沿用當前狀態即展開，不需強制。
- **房主換頁、觀眾被導頁**：minimized 隨各自分頁的 TabStorage / sessionStorage 傳遞，**不經 URL**，因此房主的收合狀態不會傳染給觀眾。
- **退出房間（房主或觀眾）**：清掉 carried 記憶；之後的頁面回到「不在房間 → 純設定」。當前畫面不強制改變。

---

## 3. 設計

### 3.1 單一真相
面板狀態的單一真相是 `this.minimized`（bool）。面板展開 ⇔ `!minimized`。

### 3.2 三個儲存概念

| 概念 | 儲存位置 | 用途 | 範圍 |
|---|---|---|---|
| `MinimiseDefault` 鏡像 | `localStorage["VideoTogetherMinimiseDefault"]`（新增） | 讓 `Init()` 能**同步**讀到設定，不必等非同步訊息 | 每網域 |
| `VideoTogetherMinimized` | TabStorage（背景，`GetRoomState`）+ sessionStorage（同源） | 在房間時跨頁繼承收/展（新增 key） | 每分頁的房間會話 |
| ~~`VideoTogetherMinimizedHere`~~ | ~~localStorage~~ | **淘汰** | — |

> minimized 是「每個用戶端自己的 UI 偏好」，只走自己分頁的 TabStorage + sessionStorage，**絕不放進 URL handoff（`linkWithMemberState`）**。

### 3.3 決策演算法

核心原則：**在「確定該展開」之前，絕不先展開。** HTML 預設已是收合，只要拿掉同步路徑的搶先展開，「展→收」閃爍即消失。

**步驟 1 — 建面板 `Init()`（同步、不存檔）：**
```
若 sessionStorage 有新鮮房間（房名非空且 timestamp 未過期）:
    在房間 → 套用 sessionStorage 的 VideoTogetherMinimized；缺失則「展開」
否則:
    不在房間 → 讀 localStorage 的 MinimiseDefault 鏡像:
        開 / 未知 → 收合（安全：絕不先展開）
        關        → 展開
```

**步驟 2 — 首次 `SyncStorageValue`（非同步、權威值）：**
此時才有真正的 `MinimiseDefault` 與 TabStorage 房間狀態。
```
RecoveryState() 判斷是否在房間（TabStorage 優先，其次 URL / sessionStorage）:
    在房間 → 套用 carried VideoTogetherMinimized；缺失則「展開」（可能由收→展，可接受）
    不在房間 → this.minimized = MinimiseDefault（權威）
同時把 MinimiseDefault 寫入 localStorage 鏡像，供下次載入同步使用。
```

因步驟 1 已用鏡像做出無閃的猜測，步驟 2 通常結果相同、不變動；若鏡像缺失或過期，步驟 2 修正之，且方向僅可能是「收→展」，永不「展→收」。

### 3.4 行為對照（消除的閃爍）
- 設定「開」（多數情況）：未知時預設收合 = 正確 → **完全零閃**。
- 設定「關」：某網域第一次載入、鏡像尚未建立 → 先收合、sync 後展開（收→展一次）；之後該網域同步正確、不再閃。
- 任何情況都**不會**再出現「展開→收合」。

---

## 4. 程式變更點

1. **`pannel.html:2`**：維持 `display:none`（已是收合預設，無需改）。

2. **新增鏡像寫入**：在 `SyncStorageValue` 處理（`vt.js:2949` 起）把 `data.MinimiseDefault` 寫入 `localStorage["VideoTogetherMinimiseDefault"]`。

3. **改寫 `Init()`（`vt.js:1864-1871`）**：依 3.3 步驟 1 的演算法決定初始 `minimized`，以 `isDefault=true` 套用（不存檔）。移除對 `VideoTogetherMinimizedHere` 的讀取。

4. **改寫 firstSync 決策（`vt.js:2976-2985`）**：依 3.3 步驟 2；不在房間才套設定，在房間交由 RecoveryState。

5. **`InRoom()`（`vt.js:1877`）**：移除無條件 `Maximize()`。改為不主動改變 `minimized`；收/展由決策演算法或 RecoveryState 決定。

6. **`RecoveryState()` / `RecoveryStateFrom()`（`vt.js:3265-3298`）**：讀取 carried `VideoTogetherMinimized`，在還原房間時設定 `this.minimized`（缺失 → 展開），再 `InRoom()`。

7. **`GetRoomState()`（`vt.js:3760`）**：回傳物件新增 `VideoTogetherMinimized: this.minimized ? 1 : 0`。

8. **`SaveStateToSessionStorageWhenSameOrigin()`（`vt.js:3784`）**：寫入 `VideoTogetherMinimized`（link=="" 或同源時）。

9. **手動 `Minimize()`/`Maximize()`（`vt.js:1819-1837`，非 default）**：
   - 在房間 → 立即把新狀態寫進 TabStorage + sessionStorage（沿用既有 `GetRoomState`/`SaveState` 流程或直接寫 key），供下一頁繼承。
   - 不在房間 → 不存任何記憶（模型 A）。
   - 移除/淘汰 `SaveIsMinimized` 寫 `VideoTogetherMinimizedHere` 的舊路徑。

10. **`exitRoom()`（`vt.js:3338`）**：`window.sessionStorage.removeItem("VideoTogetherMinimized")`。TabStorage 因 `GetRoomState()` 在 `role=Null` 回傳 `{}` 已自動清除。當前面板畫面不強制改變。

11. **`autoCollapse`（`vt.js:1431-1442`）**：維持（小視窗自動收合，`isDefault=true` 不影響存檔）；確認其首次呼叫時機在面板元素參考（`vt.js:1674-1675`）指派之後，避免對 `undefined` 操作。

---

## 5. 邊界情況

- **跨網域被導頁、且只有 URL handoff**（無 TabStorage 亦無 sessionStorage 命中）：minimized 不繼承 → 回到「在房間無 carried → 展開」。可接受（務實取捨）。
- **全新網域第一次載入、設定為「關」**：鏡像未建立 → 先收合、sync 後展開（收→展一次）。
- **退房後殘留值**：功能上無害（Init 以房名非空判斷在房間），仍於 `exitRoom()` 清除以保乾淨。

---

## 6. 測試計畫

以 Chrome headless + 面板 render harness（見記憶 `anime1sync-panel-render-harness`）與手動驗證涵蓋：

1. 設定開、不在房間、刷新 → 收合，**無任何展開閃爍**。
2. 設定關、不在房間、刷新 → 展開（首訪該網域允許一次收→展）。
3. 在房間、上頁展開、換頁 → 展開。
4. 在房間、上頁收合、換頁 → 收合。
5. 觀眾點邀請連結首次進房 → 展開。
6. 房主收合面板後換頁，觀眾端不受影響（不被收合）。
7. 退房後刷新 → 行為回到模型 A（純設定），無殘留。
8. 小視窗自動收合仍正常。

---

## 7. 風險與取捨

- **務實零閃**：只保證消除「展→收」；跨網域首訪/還原可能有一次「收→展」，已與使用者確認可接受。
- **淘汰每網域記憶**：不在房間時不記住手動收/展，換取行為一致與 bug 根除；如日後需要，可再加同分頁 sessionStorage 暫存，不影響本架構。
- **相容性**：殘留的舊 `VideoTogetherMinimizedHere` 不再被讀取，可忽略；不需遷移。

## 8. 審查發現與定案（2026-06-17，opus + codex 雙審）

兩份獨立審查確認：this 綁定、同步時序（RecoveryState 先於 firstSync）、SaveIsMinimized 無 re-entrancy/null、exitRoom 寫後刪、carried 型別、autoCollapse 雙跑皆正確；主目標「每次載入都閃」已根除。opus 判 SHIP、codex 判 FIX-FIRST，差異集中在以下**先天取捨**。

**殘留的「展→收」一次性邊界（兩個觸發點，同一根因）**

根因：`Init()` 用同步鏡像做樂觀判斷，權威值（真 `MinimiseDefault` / TabStorage 房間）要等非同步 firstSync。當鏡像=OFF（樂觀展開）但真相是「該收合」時，會有**單次**展→收：
1. **剛把設定關→開的那一次載入**：鏡像仍是舊的「0」。下次同步後即修正；首裝無鏡像時為收合（安全）。
2. **設定關 ＋ 在 TabStorage 房間裡收起過 ＋ 跨網域換頁**：Init 從鏡像=OFF 先展開，firstSync 由 TabStorage carried=1 收回。

**為何不採 codex 建議的「sync 前一律先收合」**：那會把**設定關、不在房間**的常見情況變成**每次載入都「收→展」**閃一下，等於把常見情境弄回有閃爍——以罕見邊界換常見回歸，不划算。此為先天取捨：Init 必須猜，猜展開→罕見展→收；猜收合→常見收→展。本實作選擇「常見情況零閃、僅罕見邊界殘留單次展→收」。

**對使用者實際設定（預設收合＝ON）零影響**：設定 ON 時 Init 一律先收合，永不先展開 → 任何情況都不會出現展→收（在房間且 carried=展開時為可接受的收→展）。上述兩個邊界僅影響「設定關（預設展開）」的使用者。

**定案：SHIP**（不改邏輯，記錄為已知單次邊界）。若日後要「保證永不展→收」，唯一完整解是「面板與圖示先藏、firstSync 後才現身」（hidden-gate），代價是設定關時面板現身略慢——此為 brainstorming 階段已被選擇放棄的方案。

**次要（不阻擋）**：`disableDefaultSize`（`vt.js` 約 1845/1855）改寫 firstSync 後已無讀取，為 dead write；移除安全但會牽動 33 個建置產物重建，列為可選清理。
