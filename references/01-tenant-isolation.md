# 維度 1：多租戶與資料隔離

**ERP 最高風險維度。** 一次失效 = A 公司看到 B 公司的成本、報價、客戶名單、薪資。這是會上新聞、會被解約、可能違反個資法的等級。

單租戶系統（只服務一家公司）跳過 1.1–1.5，但仍要做 1.6（部門／組織隔離常有相同問題）。

---

## 1.1 租戶識別的來源 — P0

**唯一可接受的來源是伺服器端 session／JWT claim。**

紅旗：租戶識別碼來自 client 可控處
- `req.body.company_id`、`req.query.tenant_id`、`searchParams.get('org')`
- HTTP header（`X-Tenant-Id`）而未經簽章驗證
- localStorage／cookie 中未簽章的值
- 前端傳來後只做「存在性檢查」而不驗證與當前使用者的歸屬關係

**檢查方法**：grep 租戶欄位名稱（`tenant_id`、`company_id`、`org_id`、`workspace_id`、`shop_id`），逐一確認賦值來源。任何一處來自請求參數即為 P0。

**觸發情境**：登入使用者把請求中的 `company_id` 從 3 改成 4，取得另一間公司的資料。

---

## 1.2 查詢過濾的完整性 — P0

紅旗：
- 資料存取層沒有統一的租戶過濾機制，靠每支查詢自己記得加 `where tenant_id = ?`
- 存在 raw SQL／`$queryRaw`／`.rpc()`／`knex.raw()` 且未帶租戶條件
- 聚合、統計、報表查詢繞過了一般的 repository 層
- `count()`、`exists()`、`findFirst()` 這類「只是檢查一下」的查詢漏掉過濾（常被忽略，但可用來枚舉他人資料是否存在）

**檢查方法**：先找有沒有全域機制（Prisma extension／middleware、ORM global scope、DB RLS）。
- 有 → 找繞過點：raw query、原生 driver、background job、migration script、admin 工具
- 沒有 → 抽樣 10 支查詢，統計漏加比例。漏 1 支即 P0，因為代表機制是「靠記得」

---

## 1.3 資料庫層強制隔離（RLS） — P0 / P1

應用層過濾是第一道防線，DB 層是最後一道。ERP 建議兩層都要。

紅旗：
- Postgres 未啟用 RLS，或啟用了但 policy 是 `using (true)`
- 應用程式使用 superuser／`service_role`／`postgres` 連線（會繞過所有 RLS）
- policy 依賴的 session 變數（`current_setting('app.tenant_id')`）沒有在每次連線／每個交易確實設定，或在連線池環境下會殘留上一個請求的值 ← **連線池 + session 變數是經典陷阱**

**未檢測情境**：無法從程式碼確認雲端 DB 的實際 policy 狀態時，列入「未檢測」，並給使用者 SQL 自查語句（見 `stacks/postgres-prisma.md`）。

---

## 1.4 非請求路徑的租戶上下文 — P1

這些路徑沒有「當前使用者」，最容易被遺忘：

- **排程工作／cron**：批次結帳、庫存重算、報表寄送——是否逐租戶執行且帶對上下文
- **Webhook 接收**：金流、電子發票、EDI 回呼——如何判定屬於哪個租戶？可否偽造？
- **佇列／背景工作**：job payload 是否攜帶 tenant_id，consumer 是否驗證
- **Server-Sent Events／WebSocket**：訂閱時是否驗證租戶，推播時是否過濾
- **管理後台／內部工具**：常刻意跨租戶，但是否有獨立的權限與稽核

**觸發情境**：夜間結帳工作用第一個租戶的上下文跑完全部資料，把 A 公司的成本寫進 B 公司的帳。

---

## 1.5 快取與衍生資料的租戶污染 — P1

非常陰險，因為程式碼看起來完全正確。

紅旗：
- 快取 key 不含租戶識別（`cache.set('products', ...)`）
- Next.js `unstable_cache`／`revalidateTag` 的 tag 不含租戶
- 記憶體內的模組層級變數存放使用者資料（serverless 實例會被重用）
- 全站靜態生成（SSG）用到了租戶資料
- CDN 快取了帶有租戶資料的回應而未設 `Cache-Control: private`
- 搜尋索引（Algolia／Meilisearch／pgvector）未分租戶或未在查詢時過濾

---

## 1.6 檔案與附件隔離 — P0

ERP 有大量附件：報價單 PDF、發票、合約、員工證件。

紅旗：
- 儲存路徑不含租戶（`uploads/{filename}`）→ 檔名碰撞與遍歷
- 檔案 URL 是可猜測的公開連結（流水號、原始檔名）
- Storage bucket 設為 public read
- 下載端點只檢查登入、不檢查檔案歸屬
- 簽章 URL 有效期過長（>1 小時）或無限期
- 路徑穿越：`?file=../../other-tenant/salary.pdf`

**檢查方法**：找上傳與下載兩端。上傳看路徑組成，下載看授權判斷。兩者都要驗證「這個檔案屬於當前租戶」。

---

## 1.7 跨租戶功能的刻意設計 — P1

有些 ERP 功能天生跨租戶：集團合併報表、母子公司、經銷商查看上游庫存、系統管理員支援。

檢查：
- 這些功能是否明確定義了可存取的租戶集合（而非「關掉過濾」）
- 是否有獨立稽核紀錄
- impersonation（以使用者身分登入）功能是否記錄「誰扮演了誰、做了什麼」
- 離開 impersonation 後 session 是否確實還原

---

## 常見誤判，不要報

- 系統層級的共用主檔（國家、幣別、稅率、單位）本來就不分租戶
- 明確標示為 super admin 的路由，若已有獨立驗證與稽核
- 測試檔、seed script 中的固定 tenant id
- 尚未接上路由的元件或死碼（先確認可達性）
