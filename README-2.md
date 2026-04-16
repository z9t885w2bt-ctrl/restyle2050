# 不完美也沒關係 × Restyle 2050 — 部署指南（Email + Password 版）

這版已經改成：

- 會員註冊：**姓名 + Email + 密碼**
- 會員登入：**Email + 密碼**
- **不使用手機、不使用簡訊、不使用 OTP 驗證碼**
- 保留每日每店上限，**台灣時間午夜後同店可重新累計**

---

## 檔案清單

```text
index.html           主網站（前端頁面 + JS）
supabase_schema.sql  資料庫 schema + RLS + cron + 掃碼 function
README.md            本文件
```

---

## 技術架構

```text
使用者手機
  │  掃 QR Code（帶 ?scan=cafe-a-daan）
  ↓
index.html（靜態網站，可部署於 Netlify / Vercel / GitHub Pages）
  │  Supabase JS SDK
  ↓
Supabase（PostgreSQL + Auth）
  ├── auth.users           登入帳號（Email + Password）
  ├── members              會員資料
  ├── locations            咖啡廳據點
  ├── scan_logs            掃碼紀錄
  ├── daily_scan_limits    每日每店上限
  └── draw_logs            抽獎紀錄
```

---

## Step 1 — 建立 Supabase 專案

1. 前往 Supabase 建立新專案
2. 記下：
   - **Project URL**
   - **anon public key**
3. 打開 SQL Editor
4. 貼上 `supabase_schema.sql` 全部內容並執行

> 請確認你放到前端的是 **anon public key**，不要放 `service_role` key。

---

## Step 2 — Email / Password 登入設定

Supabase Dashboard → **Authentication** → **Providers** → **Email**

確認 Email provider 已開啟。

### 可選：關掉 Email 確認信
如果你想讓使用者註冊後立刻登入，不想收驗證信：

Authentication → Email → 關閉 **Confirm email**

這樣就會是最簡單的註冊流程。

---

## Step 3 — 啟用 pg_cron

Supabase Dashboard → **Database** → **Extensions** → 搜尋 `pg_cron` → Enable

這份 schema 內已經內建 cron job：

- UTC 16:00
- 也就是 **台灣時間午夜 00:00**

它的用途是：

- 清理舊的 `daily_scan_limits`
- 清理舊的 `scan_logs`

### 重要：午夜可重新累計的原理
真正讓同一位客人「隔天到同一間店可以重新累積」的，不是 cron 去歸零。

而是因為資料表用了：

```text
(member_id, location_id, scan_date)
```

當日期變成隔天後，系統查的是新日期，所以會自然視為新的每日次數。

也就是：

- 今天在 Café A 掃滿 3 次 → 今天不能再加
- 明天再去 Café A → 可以重新累積
- 但 `total_miles` 會保留

---

## Step 4 — 填入前端設定

打開 `index.html`，找到：

```javascript
const SUPABASE_URL  = 'https://YOUR_PROJECT_ID.supabase.co'
const SUPABASE_ANON = 'YOUR_SUPABASE_ANON_KEY'
```

把它換成你的專案值。

---

## Step 5 — 產生 QR Code

每個咖啡廳的 QR Code 對應一個 URL，例如：

| 咖啡廳 | QR Code URL |
|--------|-------------|
| Café A 大安 | `https://你的網址/?scan=cafe-a-daan` |
| Café B 信義 | `https://你的網址/?scan=cafe-b-xinyi` |
| Café C 中山 | `https://你的網址/?scan=cafe-c-zhongshan` |

`qr_code_key` 必須和 `locations` 資料表中的值一致。

如果你改了 `locations.qr_code_key`，QR Code URL 也要一起改。

---

## Step 6 — 部署

### Netlify

最簡單：

1. 登入 Netlify
2. 新建 site
3. 把 `index.html` 上傳，或直接從 GitHub 連動

### Vercel

也可以直接部署靜態站。

### GitHub Pages

也可以用，但請注意：

- `anon public key` 放前端是可以的
- 但請不要把 `service_role` key 放進 repo
- 所有安全性都要靠 **RLS** 與 **Supabase Auth**

---

## 後台管理方式

### 查看會員
Database → Table Editor → `members`

### 查看掃碼紀錄
Database → Table Editor → `scan_logs`

### 查看每日次數
Database → Table Editor → `daily_scan_limits`

### 新增店點
Database → Table Editor → `locations`

### 抽獎
可以在 SQL Editor 跑：

```sql
select id, name, email, draw_tickets
from members
where draw_tickets > 0
order by random()
limit 1;
```

---

## 安全提醒

### 可以放前端的
- Supabase Project URL
- Supabase anon public key

### 不可以放前端的
- `service_role` key
- 任何後端 secret

### 若 GitHub 曾警告你的 key
如果你確認放的是 **public / anon key**，通常不是大問題。

但你仍然應該：

- 確認沒有貼錯 `service_role`
- 之後不要把其他敏感 key 一起貼進 repo
- 保持 RLS 開啟

---

## 這版和舊版差異

舊版：
- 手機號碼
- OTP 驗證碼
- `otp_verifications`
- SMS API 串接

新版：
- Email + Password
- 用 Supabase Auth 處理登入
- 不需要手機
- 不需要 OTP
- 掃碼與每日限制邏輯保留

---

## 建議部署順序

1. 在 Supabase 新專案跑 `supabase_schema.sql`
2. 到 Authentication 開啟 Email provider
3. 決定要不要關掉 Confirm email
4. 把 `index.html` 的 URL / anon key 填好
5. 先本機打開測試註冊、登入、掃碼
6. 再部署到 Netlify / Vercel / GitHub Pages
7. 最後再做 QR Code

