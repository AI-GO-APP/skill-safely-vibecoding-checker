# 技術棧：Supabase

Supabase 的安全模型可以一句話概括：**anon key 是公開的，你的資料安全 100% 取決於 RLS。**

---

## service_role key — P0

- 只要出現在任何會送到瀏覽器的位置（`NEXT_PUBLIC_*`、client component、bundle）→ 立即 P0，等同資料庫完全開放
- service_role **繞過所有 RLS**。用它做一般查詢，等於租戶隔離全部失效
- 檢查：全專案 grep `service_role`／`SERVICE_ROLE`，確認每個使用點都在伺服器端，且確實需要繞過 RLS

**常見誤用**：AI 為了「讓查詢能動」，把 client 換成 service_role client。這會靜默地移除所有隔離。

---

## RLS 啟用狀態 — P0

檢查每張含業務資料的表：

```sql
-- 列出未啟用 RLS 的表
select schemaname, tablename, rowsecurity
from pg_tables
where schemaname = 'public' and rowsecurity = false;

-- 列出所有 policy 的實際條件
select schemaname, tablename, policyname, cmd, qual, with_check
from pg_policies
where schemaname = 'public'
order by tablename;
```

紅旗：
- `rowsecurity = false` 的業務表
- `qual` 為 `true` 的 policy（等於沒有保護）
- 只有 `SELECT` policy，缺 `INSERT`／`UPDATE`／`DELETE`（**未定義的操作預設拒絕，但若專案用 service_role 寫入，就繞過了整套設計**）
- `with_check` 未設定 → 使用者可以把資料寫成別的租戶所有
- policy 條件依賴 client 可控的欄位而非 `auth.uid()`／`auth.jwt()`

**無法連線驗證時**：列為「未檢測」，把上面兩段 SQL 交給使用者去 Supabase SQL Editor 執行並回報。

---

## Storage — P0

- bucket 是否 public
- storage policy 是否比照表格設定（很多專案設了表的 RLS，忘了 storage）
- 檔案路徑是否含租戶／使用者識別，policy 是否以路徑前綴做隔離
- 簽章 URL 的有效期

```sql
select id, name, public from storage.buckets;
```

---

## Auth

- Email 確認是否啟用（否則可用他人 Email 註冊）
- 是否允許任意人註冊——ERP 通常應改為邀請制
- JWT 中的自訂 claim（role、tenant_id）如何寫入？若可由使用者自行更新 `user_metadata` 則不可信 ← **`user_metadata` 使用者可自行修改，絕不可用於授權判斷；應使用 `app_metadata`**
- session 有效期與 refresh token 輪替
- OAuth 的 redirect URL allowlist

---

## Database Functions 與 RPC

- `security definer` 函式會以擁有者權限執行 → 繞過 RLS，必須在函式內自行檢查
- `security definer` 函式是否設定了 `search_path`（未設定可被 search_path 攻擊）
- 所有 `.rpc()` 呼叫的函式都要逐一審視

---

## Realtime

- Realtime 訂閱是否受 RLS 保護（需在 publication 與 policy 兩邊都正確設定）
- 訂閱的 channel 是否含租戶隔離

---

## Edge Functions

- 是否驗證 JWT（`verify_jwt` 設定）
- 是否使用 service_role 而未自行實作授權
- 環境變數的管理
