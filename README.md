# 班表（消防員排班 App）維護筆記

> 給「久久才回來改一次」的自己看。動手前先讀完這頁。
> 最後更新：2026-09-01

## 這是什麼

單一檔案的靜態網頁 App，沒有 build、沒有套件、沒有後端。
整個專案就三個檔案：

| 檔案 | 說明 |
| :-- | :-- |
| `index.html` | **全部的東西都在這裡**（HTML + CSS + JS，約 495 行） |
| `icon.png` | iPhone 加到主畫面時的圖示 |
| `使用方式.txt` | 目前是空檔案 |

改法：直接用編輯器開 `index.html` 改，存檔後用瀏覽器打開本機檔案就能看效果，
不需要跑任何指令或啟動 server。

## 網址與部署

同一個 repo 同時被兩個平台自動部署，push 到 `main` 之後兩邊都會在 1～3 分鐘內更新：

| 平台 | 網址 | 設定 |
| :-- | :-- | :-- |
| GitHub Pages | https://emmac-xuan.github.io/scheduale/ | 從 `main` 分支的根目錄 `/` 發布 |
| Netlify | https://gentle-granita-a00475.netlify.app/ | 站名 `gentle-granita-a00475`，publish 目錄也是根目錄 |

GitHub repo：https://github.com/EmmaC-xuan/scheduale
本機路徑：`~/Coding/scheduale`

## ⚠️ 踩過的坑：`index.html` 不能離開根目錄

**兩個平台都是從 repo 根目錄發布，所以 `index.html` 必須放在根目錄。**

2026-08-23 曾經把 `index.html` 和 `icon.png` 搬進 `schedule/` 子資料夾整理，
結果隔天兩個首頁網址全部變 404 —— 網頁其實沒壞，只是根目錄沒有 `index.html`
可以當首頁。修法就是搬回根目錄（commit `363efe2`）。

**結論：不要幫這個 repo「整理資料夾結構」。** 想加檔案可以，但 `index.html`
要留在原地。

## 修改流程（照這個順序做）

```bash
cd ~/Coding/scheduale

# 1. 一定要先 pull ── 過去有在別台機器 / GitHub 網頁上直接改的紀錄
git pull

# 2. 改 index.html

# 3. commit + push（push 到 main 就等於上線）
git add -A
git commit -m "改了什麼"
git push origin main

# 4. 等 1～3 分鐘，實測兩個網址都要回 200
curl -s -o /dev/null -w "pages=%{http_code}\n" https://emmac-xuan.github.io/scheduale/
curl -s -o /dev/null -w "netlify=%{http_code}\n" https://gentle-granita-a00475.netlify.app/
```

第 1 步的 `git pull` 不要跳過。這個 App 常常是在手機上發現問題、
直接在 GitHub 網頁上改掉，本機這份很可能是舊的。

第 4 步不要只看狀態碼就收工，**用手機實際開一次網頁看畫面**，
因為所有邏輯都在前端，JS 寫錯不會讓 HTTP 變成非 200。

## 常見修改任務在哪裡改

都在 `index.html` 的 `<script>` 區塊開頭（約第 200～230 行），用搜尋字串找最快：

| 想改什麼 | 搜尋這個 | 說明 |
| :-- | :-- | :-- |
| 新增/修正國定假日 | `var HD={` | 一個 `'YYYY-MM-DD':'假日名稱'` 的表，每年要手動補新年度的假日 |
| 各班別的工時 | `DEF_HRS` | `sh` 上班 22、`ro` 輪休 0、`cp` 補休 12、`on` 外宿 12、`lv` 特休 0、`of` 公假 12、`do` 休假 8 |
| 加班時數基準 | `BASE_OT` | 預設 83 |
| 排班循環的起算日 | `var ANC` | 預設 `{y:2026,m:5,d:2}`，四天一循環（`phase()`：前 2 天上班、後 2 天輪休） |
| 班別 / 勤務的名稱 | `const STS` / `const TKS` | `STS` 是七種班別，`TKS` 是 91班 / 92班 / 值宿 |
| 顏色 | 檔案開頭 `<style>` | 主題色 `#E6AA14`（黃），國定假日紅點 `#E53935` |

注意：`ANC`、`HRS`、`BASE_OT` 這三項如果使用者在 App 的設定畫面改過，
會存在瀏覽器裡並蓋掉程式碼裡的預設值。所以改預設值對已經在用的手機**沒有效果**，
那台手機要去設定畫面自己改。

## 資料存在哪裡（重要）

**所有排班資料都存在使用者自己手機的瀏覽器 `localStorage`，沒有雲端、沒有後端。**

key 的格式：

- `fsc-anchor` — 排班起算日
- `fsc-hrs` — 各班別工時
- `fsc-base-ot` — 加班基準
- `fsc-2026-9` — 2026 年 9 月的手動調整（`fsc-` + 年 + `-` + 月，每個月一筆）

這代表：

- 改網頁、重新部署**不會**清掉資料（key 不變就還在）
- 但換手機、換瀏覽器、清瀏覽器資料 → **資料全部消失，無法救回**
- App 裡的「📤 匯出備份 / 📥 選檔還原」就是為了這件事做的，
  會把所有 `fsc-` 開頭的 key 打包成一個 JSON 檔（`schedule-backup-YYYYMMDD.json`）

**動這個 App 之前，先在手機上按一次「匯出備份」。**
尤其是如果哪天要改動 key 的命名規則（`stK()` 函式），舊資料會全部讀不到，
一定要先備份、並想好轉換舊 key 的方法。

## 驗證清單（改完 push 前後各跑一次）

- [ ] `index.html` 還在根目錄
- [ ] 本機用瀏覽器開 `index.html`，切換月份、點日期改班別都正常
- [ ] 瀏覽器 Console 沒有紅字錯誤
- [ ] push 後兩個網址都回 200
- [ ] 用手機實際開網頁看畫面，不是只看狀態碼
- [ ] 如果改過假日表：翻到那個月，確認紅點出現在對的日子
