# 技術棧：PostgreSQL / Prisma / Drizzle

---

## 連線身分 — P0

- 應用程式使用的 DB 角色是否為 superuser／table owner
  → **owner 與 superuser 預設繞過 RLS**（除非表設定 `FORCE ROW LEVEL SECURITY`）
- ERP 應為應用程式建立最小權限角色，並對業務表加上：

```sql
alter table orders enable row level security;
alter table orders force row level security;
```

- 檢查目前連線身分與權限：

```sql
select current_user, session_user;
select rolname, rolsuper, rolbypassrls from pg_roles where rolname = current_user;
```

---

## 連線池 + session 變數 = 陷阱 — P0

若 RLS policy 依賴 `current_setting('app.tenant_id')`：

- 連線池（PgBouncer、Prisma pool、serverless）會重用連線
- 用 `SET` 設定的變數會殘留到下一個請求 → **A 租戶的請求讀到 B 租戶的上下文**
- 必須使用 `SET LOCAL`（僅限當前交易），且每個查詢都包在交易中

檢查：grep `set_config`／`SET app.`，確認是否為 `SET LOCAL` 且在交易內。

---

## Prisma

- `$queryRawUnsafe`／`$executeRawUnsafe` → 逐一檢視，字串拼接即 P0
- `$queryRaw` 使用樣板字串是參數化的（安全），但若先組字串再傳入就不是
- **Prisma 預設繞過 RLS**（因為使用 owner 連線）→ 隔離只能靠應用層，此時應用層的完整性要求提高到 P0 等級
- 若使用 Prisma extension／middleware 做全域租戶過濾：確認 raw query、`$transaction` 內的原生語句、以及 `executeRaw` 是否被涵蓋
- `select` 未指定時回傳全部欄位 → 敏感欄位外洩（維度 6.1）
- schema 中金額欄位型別：`Float` → P1，應為 `Decimal`
- 關聯的 `onDelete: Cascade` 用在業務單據上 → 誤刪主檔時連帶刪除歷史（維度 3.6）

---

## Drizzle

- `sql` 樣板是參數化的；`sql.raw()` 不是 → 逐一檢視
- 動態欄位／排序欄位必須白名單，不能參數化

---

## Schema 層面檢查

```sql
-- 金額欄位型別（找出 float/double）
select table_name, column_name, data_type
from information_schema.columns
where data_type in ('double precision','real')
order by table_name;

-- 缺少唯一約束的業務鍵（人工比對單號、統編等欄位）
select conrelid::regclass as table_name, conname, contype
from pg_constraint where contype in ('u','p') order by 1;

-- 缺索引的外鍵（效能與鎖競爭）
select conrelid::regclass, conname from pg_constraint where contype = 'f';
```

其他：
- 金額 `numeric(19,4)`、數量 `numeric`，不用 float
- 時間欄位使用 `timestamptz` 而非 `timestamp`（無時區的時間在跨時區 ERP 是災難）
- 業務單據表是否有 `deleted_at` 而非硬刪
- audit log 表是否有阻擋 UPDATE／DELETE 的觸發器

---

## 遷移與存取

- migration 檔是否納入版控、是否有破壞性變更未經審查
- DB 是否對公網開放（檢查雲端防火牆規則——通常無法從程式碼確認，列「未檢測」）
- 是否啟用 `log_statement` 記錄了含個資的 SQL
