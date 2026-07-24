# 維度 7：秘密與供應鏈

Vibe coding 最高頻的 P0 來源。AI 為了讓程式碼「能跑」，很自然會把 key 寫進檔案。

**處理原則**：發現硬編碼秘密時，報告只寫位置與類型，**永遠不要把值寫進報告或對話**。並一律建議視為已洩漏、立即輪替——即使 repo 是私有的。

---

## 7.1 程式碼中的秘密 — P0

搜尋目標：
- API key、token、密碼、連線字串、私鑰（`-----BEGIN`）
- 常見前綴：`sk-`、`sk_live_`、`ghp_`、`gho_`、`xoxb-`、`AKIA`、`AIza`、`eyJ`（JWT）
- `service_role`、`SUPABASE_SERVICE`、`ADMIN_KEY`、`SECRET`
- 高熵字串（長度 >20 的隨機字元）指派給名稱含 key／token／secret／password 的變數
- 資料庫連線字串含密碼

**必查位置**（AI 特別容易寫進去的地方）：
- 設定檔、初始化程式碼、`lib/db.ts`、`lib/supabase.ts`
- 測試檔與 seed script
- 被註解掉的程式碼
- README 與文件中的「範例」（常是真的 key）
- CI 設定檔
- 前端環境變數（見維度 6.5）

---

## 7.2 版本控制歷史 — P0

**重點：從程式碼移除不等於安全。** git 歷史仍有紀錄，GitHub 的 fork 與快取也仍有。

檢查：
- `.env`、`.env.local`、`.env.production` 是否曾被提交（`git log --all --full-history -- .env*`）
- `.gitignore` 是否涵蓋所有環境檔、憑證檔（`*.pem`、`*.key`、`*.p12`）、備份檔、`.vscode/settings.json`
- 歷史 commit 中的秘密（若工具可用可掃全歷史；不可用則列為「未檢測」並給指令）
- repo 是否曾經公開過

**若確認秘密曾進入歷史**：報告必須明說「移除檔案不夠，必須輪替該憑證」。

---

## 7.3 秘密管理機制 — P1

- 是否使用平台的 secret 管理（Vercel／Zeabur／GCP Secret Manager），而非檔案
- 正式與測試環境是否使用不同的憑證 ← 常見錯誤：測試環境用正式 DB
- 是否有輪替機制與紀錄
- 團隊成員離職後憑證是否輪替
- 是否有最小權限的憑證（用 service_role 做所有事 = 沒有權限概念）

---

## 7.4 幻覺套件與名稱搶註 — P0

**AI 特有風險。** AI 會產生不存在的套件名稱；攻擊者預先註冊這些名稱並植入惡意程式碼（slopsquatting）。

檢查 `package.json` 中每個依賴：
- 套件在 registry 上是否存在
- 週下載量是否極低（<1000 且非知名小工具 → 可疑）
- 發布時間是否很近、版本數是否只有一兩個
- 名稱是否與知名套件相近（`lodahs`、`reqeusts`、`crossenv`）
- 維護者是否不明

**若無法連網查證**，列為「未檢測」並列出可疑名單請使用者自行確認。

---

## 7.5 依賴健康度 — P1

- lockfile 是否存在且已提交（否則每次安裝版本都可能不同）
- 已知漏洞：可執行 `npm audit --omit=dev`（npm 自帶，無需安裝）
- 是否有長期未維護的關鍵依賴
- 是否安裝了不必要的高風險套件
- `postinstall` 腳本：安裝時會執行任意程式碼，檢查有無來路不明者
- 直接依賴 git URL 或未鎖版本的來源

---

## 7.6 前端第三方腳本 — P1

- 從 CDN 載入的 script 是否有 SRI（`integrity` 屬性）
- 是否引入了不必要的第三方腳本（每一個都能讀取頁面上的 ERP 資料）
- GTM 等標籤管理工具的存取權限控管（能注入任意 JS 到你的系統）

---

## 7.7 CI/CD 與部署 — P1

- CI 設定中是否有明文秘密
- workflow 是否對 fork 的 PR 暴露 secret（`pull_request_target` 風險）
- 部署權限是否過寬（任何人可推 main 即可部署到正式）
- 建置產物是否包含 `.env` 或 source map
- 是否有分支保護與必要審查

---

## 常見誤判，不要報

- 公開的可發布金鑰（Supabase anon key、Stripe publishable key、Google Maps API key）——但仍要確認：anon key 的安全性完全依賴 RLS 是否啟用（回到維度 1.3），Maps key 是否設定了網域限制
- 測試用的假值（`test_key_123`、`xxxxx`）
- 範例檔 `.env.example` 中的佔位符
- 已註明為公開示範用途的憑證
