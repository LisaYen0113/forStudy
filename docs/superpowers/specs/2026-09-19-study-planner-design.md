# forStudy — 研究所讀書排程器

**設計規格**

| 項目 | 內容 |
|---|---|
| 日期 | 2026-09-19 |
| 狀態 | 設計已確認，待撰寫實作計畫 |
| 遠端 repo | https://github.com/LisaYen0113/forStudy.git |

---

## 1. 問題陳述

研究所一學期修了六門課，加上課外的閱讀與專案，很容易發生「某一科整整兩週沒碰」而自己沒察覺。既有的行事曆工具（Google Calendar 等）以「時間」為單位，但要回答「這週六科我都讀過了嗎」需要人工比對，沒有強制力。

**核心痛點不是「忘記什麼時候要讀書」，而是「不確定自己這週有沒有漏掉某一科」。**

## 2. 目標與成功標準

**目標**：把學校課表變成週曆底圖，使用者在空白節次格安排「要讀／想做的事」，每週自動重複，並在畫面上方明確顯示每一項本週是否達標。

**成功標準**：

1. 打開網頁 3 秒內能回答「這週我還缺哪一科沒讀」
2. 排定一次任務後，之後每週不必重新排
3. 匯入課表後，上課時段自動佔位，無法被誤排任務
4. 手機與電腦看到同一份資料

**非目標**：不追求取代 Google Calendar（沒有真正的時間軸、沒有提醒通知、不與外部行事曆同步）。

## 3. 範圍

### 3.1 範圍內

- Email + 密碼登入（Supabase Auth），關閉信箱驗證以避開寄信速率限制
- 部署至 Vercel，透過環境變數連接 Supabase
- 上傳課表截圖 → 瀏覽器本地 OCR → 可編輯的候選課程表格 → 確認匯入
- 內建課表範本可一鍵載入（OCR 失敗時的保底）。公開版為虛構假資料，真實課表存於本機未進版控的檔案，詳見 §6.2
- 節次制週曆（縱軸 `D1~D8` + `E0~E4` 共 13 列，橫軸週一至週日）
- 事項管理（自由命名，分「科目」/「課外」兩類，可設每週目標次數）
- 點擊空白格指派事項，每週自動重複
- 點擊任務格打勾完成
- 每週達成率面板，未達標項目紅字警示
- 前後週瀏覽（歷史紀錄保留）
- 節次時間標籤自訂
- 課表課程管理（檢視／編輯／停用／刪除）

### 3.2 範圍外（明確不做）

| 項目 | 理由 |
|---|---|
| 推播／Email 提醒通知 | 需要背景服務與通知權限，V1 先靠「打開就看到紅字」 |
| 多學期切換 | 學期交界時重新匯入課表即可，尚不需要版本化 |
| 統計圖表（趨勢、熱圖） | 先確認每日使用習慣再決定要什麼圖 |
| 拖放調整排程 | 點擊指派在手機上更可靠，且少一個依賴 |
| 匯出 Excel / iCal | 尚無需求 |
| 離線模式 / PWA | OCR 本來就需要網路下載語言模型 |
| 任務未達標的原因紀錄 | 尚無需求 |
| 不在格子裡的「臨時 +1」 | 打勾一律對應到某個格子，維持資料模型單純 |

### 3.3 範圍升級條件

若下列情況發生，這個專案需要拆分成子專案重新設計：

- 需要多人協作（同學共用同一份課表）
- 需要多學期並存與跨學期統計
- OCR 需要支援照片（非截圖）與多種課表格式

## 4. 名詞定義

| 名詞 | 定義 |
|---|---|
| **節次** | 時間單位代碼，`D1`~`D8`（日間）與 `E0`~`E4`（夜間），共 13 個。順序固定，見 `PERIOD_CODES` |
| **課程** | 來自學校課表的固定行程。佔週曆格子、不可編輯內容、不列入達成率 |
| **事項** | 使用者自己定義的「要讀／想做的事」，可為研究所科目或課外事務。有每週目標次數 |
| **排程** | 把某個事項指派到某個「星期 + 節次」的格子，每週重複 |
| **完成** | 某一週的某個格子被實際打勾的紀錄 |
| **週** | 週一 00:00 至週日 23:59（本地時區 Asia/Taipei）。以 `week_start`（該週週一的 `YYYY-MM-DD`）識別 |

## 5. 使用者流程

### 5.1 首次使用

1. 開啟網頁 → 顯示登入頁 → 切到「註冊」→ 輸入 email 與密碼
2. 註冊完成立即登入（**不寄驗證信**，見 §6.1）
3. 系統自動為此帳號建立 `settings` 資料列
4. 畫面提示「尚未有課表」→ 兩個選項：
   - **上傳課表截圖**（進入 5.2）
   - **一鍵載入課表範本**（直接帶入範本中的課程）
5. 建立事項（系統在匯入課表後，自動依課程建立同名「科目」事項，目標次數預設 1）
6. 點空白格指派事項到週曆
7. 開始使用

### 5.2 匯入課表截圖

1. 點「匯入課表」→ 對話框
2. 選擇／拖入截圖 → 顯示預覽（可縮放平移）
3. 按「開始辨識」→ 動態載入 Tesseract.js（首次約 10–20 MB 語言模型，顯示下載與辨識進度）
4. 取得 word-level bbox → `groupWordsIntoRows` → `parseScheduleText` → 候選課程陣列
5. **並排顯示**：左為原始截圖，右為可編輯表格
   - 每列欄位：納入 ☑ / 科目名稱 / 星期 / 起節次 / 迄節次 / 教室 / 教師 / 學分
   - 辨識信心低的欄位以黃底標示
   - 可增列、刪列、直接改任一欄位
6. 使用者確認 → 寫入 `courses` 表 → 週曆立即更新
7. 同時為每個納入的課程建立同名「科目」事項（若同名事項已存在則沿用）

**替代路徑**：對話框內固定有「載入課表範本」按鈕，把範本資料填入同一張可編輯表格，使用者確認後匯入。此路徑完全不需 OCR。

### 5.3 每日／每週使用

1. 開啟網頁 → 直接進入週曆（本週）
2. 上方面板顯示本週達成狀況
3. 讀完某項 → 點對應格子 → 打勾
4. 想調整排程 → 點格子 → 換事項或清除
5. 想回顧 → 週導覽列 `‹ 上週` / `本週 ›`

## 6. 功能規格

### 6.1 登入

採 Email + 密碼，**不使用**魔法連結。

**為何不用魔法連結**：Supabase 內建寄信服務限制為**每小時 2 封**（見 [Supabase 速率限制文件](https://supabase.com/docs/guides/auth/rate-limits)），開發測試期極易卡住。改用密碼登入後，登入完全不寄信。

**必須關閉信箱驗證**：Supabase 預設註冊時要求點擊驗證信，同樣會消耗寄信額度。設定路徑 `Authentication → Sign In / Providers → Email → Confirm email` 關閉。關閉後註冊即完成登入，全程不寄任何信。

| 流程 | Supabase API | 是否寄信 |
|---|---|---|
| 註冊 | `signUp({ email, password })` | 否（已關閉驗證） |
| 登入 | `signInWithPassword({ email, password })` | 否 |
| 登出 | `signOut()` | 否 |
| 忘記密碼 | `resetPasswordForEmail(email)` | **是**（罕用，2 封/小時足夠） |
| 重設密碼 | `updateUser({ password })` | 否 |

**規格**

- 登入頁含三個模式：登入 / 註冊 / 忘記密碼，以頁籤切換
- 密碼規則：至少 8 字元，需含英文與數字。註冊時即時顯示規則檢查結果
- 密碼輸入框提供顯示／隱藏切換
- 錯誤訊息對應：`Invalid login credentials` → 「Email 或密碼錯誤」；`User already registered` → 「這個 Email 已經註冊過」
- 註冊成功 → 直接進入主畫面，顯示一次性提示「帳號已建立」
- 未登入時顯示登入頁，其餘畫面一律重導向登入頁
- 登入狀態由 `supabase.auth.onAuthStateChange` 驅動
- 登出按鈕置於設定頁
- 忘記密碼送出後顯示「已寄出重設信，請至信箱收信」，含 60 秒冷卻

**安全性**：Supabase 內建 bcrypt 雜湊儲存密碼，前端不接觸雜湊值。密碼永不放進 localStorage 或任何前端狀態儲存，登入後僅保存 Supabase 簽發的 JWT session。

### 6.2 課表匯入

**OCR 執行規格**

- 套件：`tesseract.js`（動態 `import()`，不進主 bundle）
- 語言：`chi_tra+eng`，語言檔自 CDN 載入（`https://tessdata.projectnaptha.com/4.0.0`）
- 取用 word-level bbox：`recognize(image, lang, { blocks: true })` 的 `data.blocks[].paragraphs[].lines[].words[]`
- 每個 word 保留 `text`、`bbox`（`x0,y0,x1,y1`）、`confidence`

**`groupWordsIntoRows` 分群規則**

- 依 `y` 中心排序所有 word
- 遍歷時，若當前 word 的 y 中心與當前群組的「平均字高 × 0.6」以內，歸入同群；否則開新群
- 使用相對閾值（相對於字高）而非絕對像素，以適應不同截圖解析度
- 每群內依 `x0` 升序排序，串接成該列文字
- 輸出依 `y` 升序

**`parseScheduleText` 擷取規則**

1. **節次正規化**：`O`/`o` → `0`；`l`/`I`/`i` → `1`（僅在節次上下文內）
2. **節次擷取**：正則 `/[DEde]\s?\d{1,2}/g`，支援分隔符 `-`、`–`、`—`、`~`、`～`、`至`（可有空白）
   - 單一節次 `D5` → `start = end = 'D5'`
   - 範圍 `D5-D7` → `start = 'D5'`, `end = 'D7'`
   - 擷取結果逐一比對 `PERIOD_CODES`，無效者丟棄
3. **星期擷取**：正則 `/(?:週|星期|禮拜|拜)?\s*([一二三四五六日天ㄧ])/`
   - 對照：一→1、二→2、三→3、四→4、五→5、六→6、日／天→7
   - `ㄧ`（注音）視為 `一`
4. **欄位黑名單**（用於剔除，不當作名稱）：
   - 選課標記：`必`、`選`、`通`、`全`
   - 表頭與欄名：`NO`、`學年度`、`學期`、`課程代碼`、`開課單位`、`科目名稱`、`學分`、`授課教師`、`教室`、`備註`、`通識領域`、`節次`、`星期`、`週別`、`學生選課設定`
   - 樣式比對：`[A-Z]{2,3}\d{3,4}`（教室，如 `AB201`）、`[A-Z]\d{6,}`（課程代碼）、純數字、純英數
5. **名稱擷取**：剔除上述欄位後，取該列**最長的中文連續片段**（允許全形括號與英數字穿插）
6. **信心標示**：名稱長度 < 2、或找不到星期、或找不到節次 → 該欄位標為低信心（UI 顯示黃底）
7. 輸出 `ParsedCourse[]`，每項含 `name`、`dayOfWeek`、`startPeriod`、`endPeriod`、`location`、`teacher`、`rawLine`、`lowConfidenceFields`

**範本機制（本機真實資料與公開版假資料分離）**

此 repo 為公開，因此**真實課表資料不得進入版控**。範本設計為兩層：

| 檔案 | 進版控 | 內容 |
|---|---|---|
| `src/features/import/courseTemplate.fake.ts` | ✅ 是 | 示範用假資料，供測試、Storybook 與公開展示 |
| `src/features/import/courseTemplate.local.ts` | ❌ 否（`.gitignore`） | 使用者本人的真實課表，僅存在本機 |

解析順序由 `resolveCourseTemplate()` 處理：

```ts
const localModules = import.meta.glob<{ default: CourseTemplateEntry[] }>(
  './courseTemplate.local.ts',
  { eager: true },
)
export function resolveCourseTemplate(): CourseTemplateEntry[] {
  const local = Object.values(localModules).at(0)?.default
  return local ?? FAKE_COURSE_TEMPLATE
}
```

- 使用 `import.meta.glob` 而非靜態 `import`，因為本機檔案不存在時靜態 import 會讓建置失敗；`import.meta.glob` 找不到檔案時回傳空物件
- 首次 clone 此 repo 的人（或 CI）自動取得假資料，建置不會壞
- `.env.local` 與 `courseTemplate.local.ts` 皆列入 `.gitignore`

**公開版示範資料（`courseTemplate.fake.ts`，全部為虛構）**

| 科目名稱 | 星期 | 節次 | 教室 | 教師 | 學分 | 預設納入 |
|---|---|---|---|---|---|---|
| 研究方法 | 一 | D3–D4 | AB201 | 王大明 | 2 | ✅ |
| 專題討論 | 二 | D5–D6 | CD105 | 陳怡君 | 1 | ✅ |
| 資料庫系統 | 三 | D2–D4 | EF302 | 李志豪 | 3 | ✅ |
| 書報討論 | 三 | D7–D8 | GH401 | 張雅婷 | 0 | ❌（0 學分預設排除） |
| 演算法 | 四 | D5–D7 | EF302 | 林建宏 | 3 | ✅ |
| 機器學習導論 | 五 | D5–D7 | IJ203 | 吳佩珊 | 3 | ✅ |

**匯入驗證**：

- 科目名稱必填，1–100 字元
- `startPeriod` / `endPeriod` 必須在 `PERIOD_CODES` 內
- `endPeriod` 的順序索引必須 ≥ `startPeriod`；若小於，UI 提示並自動對調
- 同一「星期 + 節次」不得與現有課程重疊 → 衝突列標紅並阻擋匯入該列

### 6.3 週曆

**版面**：CSS Grid，13 列 × 8 欄（左側節次標籤欄 + 週一至週日）

- 節次列：**固定全數顯示** `D1~D8`、`E0~E4`，不做自動隱藏
- 節次標籤：顯示 `code`；若 `settings.period_labels` 有對應值則顯示為 `"{label} {code}"`
- 欄寬：桌機等寬 7 欄；手機橫向捲動，節次標籤欄 `position: sticky` 固定於左側

**格子狀態**

| 狀態 | 表現 | 可互動 |
|---|---|---|
| 空 | 淡灰底、hover 顯示 `+` | 點擊 → 指派選單 |
| 課程 | 深色底、顯示科目名（截斷加 tooltip） | 點擊 → 課程詳情卡（唯讀） |
| 任務 | 事項色底、顯示事項名 | 點擊 → 任務選單 |
| 任務（已完成） | 同色但降低飽和度、顯示 `✓` | 點擊 → 任務選單 |

**跨節次合併**：連續且屬於同一課程／同一事項的格子，以 CSS Grid `grid-row: span n` 合併為單一區塊，避免重複顯示名稱

**重疊處理**：同一格若同時有課程與任務（資料異常或匯入衝突），以課程優先顯示，任務格加紅色邊框警示

**週導覽**：`‹ 上週` ／ `本週` ／ `下週 ›`，顯示 `YYYY/MM/DD – MM/DD`。標題列右側在非本週時顯示「回到本週」按鈕

### 6.4 事項管理

**位置**：畫面右側側欄，手機上移至週曆下方並改為橫向卡片捲動

**分區**：`科目`（category = `subject`）與 `課外`（category = `extracurricular`）兩區，各自可折疊

**每張事項卡顯示**

- 左側色塊（事項顏色）
- 名稱
- 本週進度：`已完成 / 目標`，例如 `1 / 2`
- 達標顯示 `✓`；未達標顯示 `⚠` 並以警示色標示
- 未排入週曆時額外顯示「未排時間」提示

**操作**

- 點卡片 → 展開編輯：名稱、分類、每週目標次數（1–20）、顏色、是否列入追蹤、封存、刪除
- 底部「＋ 新增事項」→ 建立表單（名稱、分類、目標次數、顏色）
- 刪除事項採兩段確認，並明確告知將一併刪除其排程與完成紀錄
- 封存：從清單與週曆隱藏，但保留歷史完成紀錄

### 6.5 排程指派

**點擊空白格** → 彈出選單：

- 上方搜尋框（過濾事項）
- 分「科目」「課外」列出所有未封存事項，標示各自當前目標次數
- 底部「＋ 新建事項」（就在此格建立並直接排入）
- 選擇事項 → 寫入 `slots`（`user_id + day_of_week + period_code` 唯一）

**點擊任務格** → 彈出選單：

- 「✓ 標記完成」／「取消完成」（切換該格本週的 completion 紀錄）
- 「改為其他事項」→ 回到事項選擇清單（既有 completion 保留）
- 「從週曆移除」→ 只刪除該 `slot`；**該格本週既有的完成紀錄保留**，歷史事實不因改排程而抹除

**限制**

- 有課程佔用的格子不顯示 `+`，點擊只顯示課程詳情
- 目標次數與排定格數不一致時（例如目標 2 次但只排了 1 格），在事項卡顯示提示，**不阻擋**

### 6.6 每週追蹤

**達成率面板**（週曆上方）

- 週別標題與範圍
- 每個「列入追蹤且未封存」的事項一顆膠囊：`色塊 名稱 已完成/目標`
- 達標 `✓`、未達標 `⚠`
- 摘要計數：`已達標 N / 總計 M 項`
- 點擊膠囊可跳至該事項詳情

**完成寫入**

- 打勾 → 新增 `completions` 紀錄（`task_id`、`week_start`、`slot_day`、`slot_period`）
- 取消 → 刪除對應紀錄
- 唯一鍵 `(user_id, task_id, week_start, slot_day, slot_period)` 保證同一格同一週不會重複計數
- `week_start` 於打勾當下依「該格所屬週」決定，因此回顧過去週次時打勾會正確記在過去那一週

**達標判定**：`該週該事項的完成筆數 >= weekly_target`

### 6.7 設定

- **節次時間標籤**：13 個節次各一輸入框，空白則只顯示代碼
- **課表課程管理**：列出所有課程（含已停用），可編輯名稱／星期／節次／教室／教師、切換停用、刪除
- **事項封存區**：列出已封存事項，可還原
- **帳號**：顯示登入 email、登出

## 7. 資料模型

### 7.1 前端常數（單一事實來源）

節次順序與合法性在前端常數定義，資料庫以 text 儲存並由 zod 驗證，不設外鍵。

```ts
export const PERIOD_CODES = [
  'D1','D2','D3','D4','D5','D6','D7','D8',
  'E0','E1','E2','E3','E4',
] as const
export type PeriodCode = typeof PERIOD_CODES[number]

export const DAYS = [
  { value: 1, label: '週一', short: '一' },
  { value: 2, label: '週二', short: '二' },
  { value: 3, label: '週三', short: '三' },
  { value: 4, label: '週四', short: '四' },
  { value: 5, label: '週五', short: '五' },
  { value: 6, label: '週六', short: '六' },
  { value: 7, label: '週日', short: '日' },
] as const
```

**為何節次不做成資料表**：節次代碼固定不變，唯一可變的是顯示標籤。用 `settings.period_labels` 的 JSONB 欄位即可，省去一張表、一個 trigger 與一次查詢。

### 7.2 Supabase (Postgres) Schema

```sql
-- ============ settings ============
create table public.settings (
  user_id       uuid primary key references auth.users(id) on delete cascade,
  period_labels jsonb not null default '{}'::jsonb,   -- {"D1": "08:10", ...}
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);

-- ============ courses：學校課表，只佔位不追蹤 ============
create table public.courses (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references auth.users(id) on delete cascade,
  name         text not null check (char_length(name) between 1 and 100),
  teacher      text not null default '',
  location     text not null default '',
  credits      numeric(4,2) not null default 0 check (credits >= 0),
  day_of_week  smallint not null check (day_of_week between 1 and 7),
  start_period text not null,
  end_period   text not null,
  color        text not null default '#1e293b',
  is_excluded  boolean not null default false,   -- true = 不顯示於週曆
  created_at   timestamptz not null default now()
);
create index courses_user_idx on public.courses (user_id);

-- ============ tasks：要讀／想做的事 ============
create table public.tasks (
  id             uuid primary key default gen_random_uuid(),
  user_id        uuid not null references auth.users(id) on delete cascade,
  title          text not null check (char_length(title) between 1 and 60),
  category       text not null default 'subject'
                 check (category in ('subject','extracurricular')),
  color          text not null default '#6366f1',
  weekly_target  smallint not null default 1 check (weekly_target between 1 and 20),
  track_progress boolean not null default true,
  sort_order     int not null default 0,
  is_archived    boolean not null default false,
  created_at     timestamptz not null default now()
);
create index tasks_user_idx on public.tasks (user_id) where is_archived = false;

-- ============ slots：每週重複的固定排程 ============
create table public.slots (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references auth.users(id) on delete cascade,
  day_of_week smallint not null check (day_of_week between 1 and 7),
  period_code text not null,
  task_id     uuid not null references public.tasks(id) on delete cascade,
  created_at  timestamptz not null default now(),
  unique (user_id, day_of_week, period_code)
);
create index slots_user_idx on public.slots (user_id);

-- ============ completions：完成紀錄 ============
create table public.completions (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references auth.users(id) on delete cascade,
  task_id      uuid not null references public.tasks(id) on delete cascade,
  week_start   date not null,
  slot_day     smallint not null check (slot_day between 1 and 7),
  slot_period  text not null,
  completed_at timestamptz not null default now(),
  unique (user_id, task_id, week_start, slot_day, slot_period)
);
create index completions_user_week_idx on public.completions (user_id, week_start);

-- ============ RLS ============
alter table public.settings    enable row level security;
alter table public.courses     enable row level security;
alter table public.tasks       enable row level security;
alter table public.slots       enable row level security;
alter table public.completions enable row level security;

create policy "settings_own"    on public.settings
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "courses_own"     on public.courses
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "tasks_own"       on public.tasks
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "slots_own"       on public.slots
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "completions_own" on public.completions
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- ============ 新使用者自動建立 settings ============
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = ''
as $$
begin
  insert into public.settings (user_id) values (new.id)
  on conflict (user_id) do nothing;
  return new;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();
```

### 7.3 串聯刪除行為

| 刪除 | 連帶影響 |
|---|---|
| `tasks` 一列 | 其 `slots` 與 `completions` 一併刪除（`on delete cascade`） |
| `courses` 一列 | 無（課程不參與追蹤） |
| `slots` 一列 | 無（`completions` 保留，歷史完成紀錄不因改排程而遺失） |
| `auth.users` 一列 | 全部資料一併刪除 |

## 8. 純函式規格（TDD 重點）

這四個函式是整個系統最容易出錯、也最值得先寫測試的部分。全部為無副作用純函式，置於 `src/domain/`。

### 8.1 `getWeekStart(date: Date): string`

回傳該日期所屬週的週一日期，格式 `YYYY-MM-DD`，以本地時區計算。

- `getDay()` 為 0（週日）時，屬於**前一週**的週一
- 回傳值僅含年月日，不含時間，避免時區偏移

### 8.2 `parsePeriodRange(text: string): { start: PeriodCode; end: PeriodCode } | null`

從一段文字擷取節次範圍。

- 正規化：`O`/`o` → `0`；`l`/`I`/`i` → `1`（僅在 `[DE]` 之後的位置）
- 支援分隔符：`-`、`–`、`—`、`~`、`～`、`至`，可有空白
- 大小寫不敏感
- 單一節次 → `start === end`
- 任一端不在 `PERIOD_CODES` 內 → 回傳 `null`
- `end` 索引小於 `start` 索引 → 自動對調
- 找不到節次 → `null`

### 8.3 `groupWordsIntoRows(words: OcrWord[]): OcrRow[]`

把 Tesseract 的 word 陣列依視覺列分群。

```ts
type OcrWord = { text: string; bbox: { x0: number; y0: number; x1: number; y1: number } }
type OcrRow  = { text: string; words: OcrWord[]; yCenter: number }
```

- 群組閾值 = 當前群組平均字高 × 0.6（相對閾值，適應不同解析度）
- 群組內依 `x0` 升序、以單一空格串接
- 輸出依 `yCenter` 升序
- 空陣列輸入 → 空陣列輸出

### 8.4 `parseScheduleText(rows: OcrRow[]): ParsedCourse[]`

```ts
type ParsedCourse = {
  name: string
  dayOfWeek: number | null
  startPeriod: PeriodCode | null
  endPeriod: PeriodCode | null
  location: string | null
  teacher: string | null
  rawLine: string
  lowConfidenceFields: string[]   // 例如 ['name', 'dayOfWeek']
}
```

- 逐列解析，規則見 §6.2
- 無法從該列取得節次**且**無法取得星期 → 該列視為表頭或雜訊，跳過
- 名稱擷取失敗 → `name` 為空字串並將 `'name'` 列入 `lowConfidenceFields`

### 8.5 `computeWeeklyProgress(tasks, completions, slots, weekStart)`

```ts
type ProgressItem = {
  taskId: string
  title: string
  category: 'subject' | 'extracurricular'
  color: string
  target: number
  completed: number
  isMet: boolean
  hasSlots: boolean
}
type WeeklyProgress = {
  weekStart: string
  items: ProgressItem[]
  metCount: number
  trackedCount: number
}
```

- 輸入：`tasks`（全部事項）、`completions`（全部或已篩選皆可，函式內自行依 `weekStart` 過濾）、`slots`（全部排程）、`weekStart`（`YYYY-MM-DD`）
- 僅納入 `trackProgress === true` 且 `isArchived === false` 的事項
- `completed` = 該事項在該週的 completion 筆數（**可大於 `target`**，如 `3 / 2`）
- `isMet` = `completed >= target`
- `hasSlots` = `slots` 中是否存在屬於該事項的排程列（用於顯示「未排時間」提示）
- 排序：`subject` 先於 `extracurricular` → 再依 `sort_order` → 再依 `title`
- `trackedCount` = 納入計算的事項數；`metCount` = 其中 `isMet` 為真者

## 9. 技術架構

### 9.1 技術選型

| 層 | 選擇 | 理由 |
|---|---|---|
| 建置 | Vite（最新穩定版） | 快、設定少、TypeScript 支援佳 |
| 框架 | React 19 + TypeScript（strict） | 排程格子互動多，元件化必要 |
| 樣式 | Tailwind CSS v4 | 週曆 grid 與響應式排版用 utility 最快；v4 免設定檔 |
| 狀態 | TanStack Query（伺服器狀態） + React state（UI 狀態） | 快取、重驗、樂觀更新皆由 Query 處理 |
| 後端 | Supabase（Postgres + Auth） | 免自建伺服器、內建 RLS、免費額度足夠 |
| OCR | tesseract.js | 瀏覽器本地執行、免金鑰 |
| 驗證 | zod | schema 與型別共用單一事實來源 |
| 測試 | Vitest + React Testing Library + jsdom | 與 Vite 同生態，設定最少 |

### 9.2 分層

```
UI 元件  ──►  hooks（useTasks / useSlots / ...）──► Repository 介面 ──► Supabase 實作
                                                        └──► 假實作（測試用）
純函式（domain/）◄── 由 UI 與 hooks 呼叫，不依賴任何 I/O
```

**關鍵設計**：所有資料存取走 `Repository` 介面，`SupabaseRepository` 為正式實作、`FakeRepository` 供測試使用。如此一來 hooks 與元件的測試完全不需要連線 Supabase，也讓 80% 覆蓋率目標在合理時間內達成。

### 9.3 專案結構

```
forStudy/
├─ index.html
├─ package.json
├─ vite.config.ts
├─ tsconfig.json
├─ .env.example                  # 只有變數名稱，值留空
├─ .gitignore
├─ vercel.json                   # SPA rewrite + 建置設定
├─ README.md                     # 環境設定逐步指南
├─ docs/superpowers/specs/
│  └─ 2026-09-19-study-planner-design.md
├─ supabase/migrations/
│  └─ 0001_init.sql
└─ src/
   ├─ main.tsx
   ├─ App.tsx
   ├─ index.css
   ├─ lib/
   │  ├─ env.ts                  # zod 驗證環境變數，缺少時給明確錯誤
   │  ├─ supabase.ts             # client 單例
   │  ├─ periods.ts              # PERIOD_CODES / DAYS 常數
   │  └─ colors.ts               # 事項配色盤
   ├─ domain/
   │  ├─ types.ts
   │  ├─ schemas.ts              # zod schemas
   │  ├─ getWeekStart.ts         ★
   │  ├─ formatWeekRange.ts
   │  ├─ parsePeriodRange.ts     ★
   │  ├─ groupWordsIntoRows.ts   ★
   │  ├─ parseScheduleText.ts    ★
   │  └─ computeWeeklyProgress.ts ★
   ├─ data/
   │  ├─ repository.ts           # 介面定義
   │  ├─ supabaseRepository.ts
   │  └─ fakeRepository.ts
   ├─ hooks/
   │  ├─ useSession.ts
   │  ├─ useCourses.ts
   │  ├─ useTasks.ts
   │  ├─ useSlots.ts
   │  ├─ useCompletions.ts
   │  └─ useSettings.ts
   ├─ features/
   │  ├─ auth/          LoginPage.tsx / AuthTabs.tsx / ForgotPasswordForm.tsx
   │  ├─ schedule/      WeeklyBoard.tsx / BoardCell.tsx / CourseBlock.tsx
   │  │                 TaskBlock.tsx / AssignPopover.tsx / WeekNav.tsx
   │  ├─ progress/      ProgressPanel.tsx / ProgressChip.tsx
   │  ├─ tasks/         TaskSidebar.tsx / TaskCard.tsx / TaskForm.tsx
   │  ├─ import/        ImportDialog.tsx / CourseEditTable.tsx / useOcr.ts
   │  │                 courseTemplate.fake.ts      # 公開假資料
   │  │                 courseTemplate.local.ts     # 真實課表，未進版控
   │  │                 resolveCourseTemplate.ts
   │  └─ settings/      SettingsPage.tsx / PeriodLabelForm.tsx / CourseAdmin.tsx
   └─ test/
      ├─ setup.ts
      └─ fixtures.ts
```

**檔案大小原則**：每個檔案目標 200–400 行，超過 800 行視為需要拆分。（依使用者全域規範）

## 10. 測試策略

**目標覆蓋率：80% 以上**（`vitest --coverage`，v8 provider）

| 層級 | 對象 | 方式 |
|---|---|---|
| 單元（重點） | `parsePeriodRange`、`groupWordsIntoRows`、`parseScheduleText`、`computeWeeklyProgress`、`getWeekStart` | 先寫測試再實作。涵蓋正常案例、OCR 錯字、邊界（空輸入、跨節次、跨週日、超額完成） |
| 單元 | zod schemas | 合法／非法輸入各測一組 |
| 元件 | `BoardCell`、`ProgressPanel`、`TaskSidebar`、`CourseEditTable` | React Testing Library，測互動與渲染結果，不測實作細節 |
| 整合 | hooks + `FakeRepository` | 驗證資料流：打勾 → completion 新增 → 進度更新 |
| 不測 | `SupabaseRepository`、`useOcr` | 需真實外部服務；改以型別與介面約束，並在 E2E 手動驗證 |

**TDD 執行順序**：每個純函式先寫失敗的測試 → 最小實作 → 重構。UI 元件在純函式穩定後才開始。

## 11. 安全性

依使用者全域安全規範逐項對應：

| 檢查項 | 做法 |
|---|---|
| 無硬編碼機密 | Supabase URL 與 anon key 只從 `.env.local` 讀取，經 `lib/env.ts` 以 zod 驗證；`.env.local` 在 `.gitignore` 內；`.env.example` 只有變數名稱 |
| 輸入驗證 | 所有寫入資料庫的資料先過 zod schema；資料庫層另有 check constraint 與 unique 約束 |
| 注入防護 | 全程使用 `@supabase/supabase-js` 的參數化查詢，無字串拼接 SQL |
| XSS 防護 | React 預設轉義；**禁止使用 `dangerouslySetInnerHTML`**；OCR 取得的文字一律視為純文字顯示 |
| 授權驗證 | 五張表全開 RLS，policy 皆為 `auth.uid() = user_id`；前端不放置 `service_role` key |
| 錯誤訊息 | `lib/env.ts` 缺變數時只說「缺少 VITE_SUPABASE_URL」，不輸出任何金鑰內容；資料庫錯誤統一轉為使用者可讀訊息 |
| 速率限制 | 登入與註冊完全不寄信，不受寄信額度（2 封/小時）影響；忘記密碼按鈕 60 秒冷卻；Supabase 另有登入嘗試次數限制 |
| 個人資料 | 課表截圖檔名加入 `.gitignore`，不進版控 |

**關於 anon key 的說明**：Vite 會把 `VITE_` 開頭的變數打包進前端 bundle，這是預期行為。Supabase 的 anon key 設計上就是公開的，真正的存取控制由 RLS 負責，因此不能因為「前端看得到」而誤認為外洩。真正**絕不可**放進前端的只有 `service_role` key。

## 12. 一次性環境設定

這部分需要你本人操作（我無法代為註冊帳號）。完整步驟會寫進 `README.md`，此處列出綱要。

### 12.1 Supabase（你已完成註冊）

1. 建立新專案，記下資料庫密碼與 Region
2. `SQL Editor` → 貼上並執行 `supabase/migrations/0001_init.sql` 全文
3. **`Authentication → Sign In / Providers → Email`**：
   - 確認 Email provider 為啟用
   - **關閉 `Confirm email`**（關鍵：否則註冊要收信，會撞上 2 封/小時的限制）
   - 密碼最短長度設為 8
4. `Authentication → URL Configuration`：
   - `Site URL`：本機開發時填 `http://localhost:5173`；部署後改填 Vercel 網址
   - `Redirect URLs`：加入 `http://localhost:5173/**` 與 `https://<你的專案>.vercel.app/**`
   - （僅「忘記密碼」的重設連結會用到此設定）
5. `Project Settings → API`：複製 `Project URL` 與 `anon public` key

### 12.2 本機開發

6. 複製 `.env.example` 為 `.env.local`，填入步驟 5 的兩個值
7. `npm install` → `npm run dev` → 開 `http://localhost:5173`
8. 首次使用：註冊帳號 → 匯入課表（或載入範本）→ 排事項

### 12.3 Vercel 部署

9. 至 vercel.com 以 GitHub 帳號登入（可直接沿用 `LisaYen0113`）
10. `Add New → Project` → 匯入 `forStudy` repo
11. Framework Preset 會自動偵測為 **Vite**；確認：
    - Build Command：`npm run build`
    - Output Directory：`dist`
    - Install Command：`npm install`
12. **`Environment Variables`** 加入與 `.env.local` 相同的兩組值：
    - `VITE_SUPABASE_URL`
    - `VITE_SUPABASE_ANON_KEY`
    - 環境勾選 Production / Preview / Development 全選
13. `Deploy` → 取得 `https://<專案>.vercel.app`
14. 回到 Supabase 步驟 4，把 Vercel 網址補進 `Site URL` 與 `Redirect URLs`

之後每次 `git push` 到 `main`，Vercel 會自動重新建置部署。

**`vercel.json`**（置於專案根目錄）：

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "framework": "vite",
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

`rewrites` 確保前端路由（若日後加入）在直接輸入網址時不會 404。

### 12.4 部署與隱私的關係

- **Vercel 網址是公開的**，任何人打開只會看到登入頁
- 所有資料受登入與 RLS 保護，未登入者無法讀取任何一列
- **Vercel 從 GitHub 建置，因此建置出的版本用的是假範本資料** — 這是正確的：真實課表只在你本機匯入一次，之後存進 Supabase，部署版讀 Supabase 就看到了
- `courseTemplate.local.ts` 不存在於 GitHub，所以 Vercel 上的人（包括未來的你）只會拿到假範本

### 12.5 備案：自訂 SMTP

若日後「忘記密碼」也嫌不夠用（或想開啟信箱驗證），可接 Resend：

1. resend.com 註冊（免費 3000 封/月）
2. 建立 API Key，驗證寄件網域（或先用測試網域）
3. Supabase `Project Settings → Auth → SMTP Settings` 填入 Resend 的 SMTP 主機、埠、帳密
4. 回到 `Authentication → Rate Limits` 調高寄信上限

## 13. 風險與緩解

| 風險 | 影響 | 緩解 |
|---|---|---|
| **中文表格 OCR 準確率不佳** | 匯入結果需大量人工修正 | 可編輯表格為主要介面（OCR 只是省打字）；內建課表範本可一鍵載入；低信心欄位黃底標示；欄位黑名單過濾表頭雜訊 |
| Tesseract 語言模型首次載入慢（10–20 MB） | 首次辨識等待時間長 | 動態 `import()`，僅在按下辨識時才載入；顯示下載與辨識雙階段進度；之後由瀏覽器快取 |
| 截圖解析度過低導致 bbox 分群失準 | OCR 全盤失敗 | 分群採相對閾值；失敗時明確提示「請改用範本或手動輸入」，不假裝成功 |
| 寄信額度僅 2 封/小時 | 註冊或登入受阻 | 已關閉信箱驗證並改用密碼登入，日常登入不寄信；僅「忘記密碼」會用到。若真的卡住，可接自訂 SMTP（Resend 免費 3000 封/月），步驟寫進 README 備案 |
| 每週排程與實際生活脫節 | 使用者放棄使用 | 未達標只是紅字提醒，不阻擋任何操作；可隨時改排程或封存事項 |

## 14. 已知限制

- 打勾只能發生在已排程的格子上；臨時多讀一次無法記錄（此為刻意設計，見 §3.2）
- 節次與日期無關聯，因此不會自動跳過國定假日或停課日
- 所有資料以單一時區（Asia/Taipei）計算週界
- 課程與事項在同一格重疊時，課程優先顯示，任務以紅框警示（正常流程下不會發生）

## 15. 視覺與文案細節

### 15.1 事項配色盤

八色，皆通過深淺底對比檢查：

| 用途 | 色碼 |
|---|---|
| 預設 | `#6366f1` |
| 備選 1 | `#0ea5e9` |
| 備選 2 | `#10b981` |
| 備選 3 | `#f59e0b` |
| 備選 4 | `#ef4444` |
| 備選 5 | `#8b5cf6` |
| 備選 6 | `#ec4899` |
| 備選 7 | `#14b8a6` |

課程格固定使用深色 `#1e293b`，與事項色明確區隔。新建事項時，顏色依建立順序自色盤輪替。

### 15.2 尺寸

| 項目 | 桌機 | 手機（< 768 px） |
|---|---|---|
| 週曆格子高度 | 48 px | 44 px |
| 節次標籤欄寬 | 64 px | 56 px |
| 頁面水平留白 | 24 px | 12 px |

響應式斷點統一使用 768 px。

### 15.3 空狀態文案

| 情境 | 文案 |
|---|---|
| 尚無課表 | 「還沒有課表。上傳截圖讓我幫你辨識，或直接用範本開始。」 |
| 尚無事項 | 「還沒有要讀的東西。新增一項，或匯入課表後由課程自動建立。」 |
| 事項未排時間 | 「尚未排入週曆，本週無法達標」 |
| 本週全數達標 | 「本週 6 項全部達標 🎉」 |
| 本週尚無完成紀錄 | 「這週還沒開始。點格子打勾就算完成。」 |

### 15.4 文案語氣

一律使用第二人稱「你」，動詞開頭，不用敬語。範例：「匯入課表」「新增事項」而非「請點此匯入您的課表」。
