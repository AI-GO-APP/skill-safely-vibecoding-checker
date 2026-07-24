# 技術棧：Next.js（App Router）

> 版本差異大，寫結論前請先確認專案的 Next.js 版本與 `node_modules/next` 內的文件，不要憑記憶斷言 API 行為。

---

## Server Action 是公開端點 — P0

最常被誤解的一點：**`'use server'` 函式會被編譯成可被任何人直接呼叫的 HTTP 端點**，不論它在 UI 上是否被權限判斷包住。

紅旗：
```tsx
{isAdmin && <form action={deleteOrder}>...</form>}   // 前端擋
```
而 `deleteOrder` 內部沒有任何 session 與權限檢查。

**每一個 server action 都必須自己驗證身分、租戶與權限。** 逐一檢查所有 `'use server'` 檔案與函式。

---

## Route Handler 授權

- `app/api/**/route.ts` 不受 layout 保護，layout 只影響頁面渲染
- middleware 的 `matcher` 是否涵蓋 API 路由——常見錯誤是只 match 頁面路徑
- middleware 用黑名單保護（`if path.startsWith('/admin')`）→ 新端點必然遺漏，應改為預設拒絕

---

## Server／Client 邊界

- `'use client'` 檔案中的任何常數都會進入 bundle → 不可放秘密
- 從 Server Component 傳給 Client Component 的 props 會序列化進 HTML（`__NEXT_DATA__`／RSC payload）→ **頁面只顯示 3 個欄位，但整列資料含成本與薪資都在原始碼裡**
- 檢查方法：對照 server 端查詢的 select 欄位與實際傳給 client 的 props

---

## 環境變數

- `NEXT_PUBLIC_*` 一律進入 client bundle。逐一確認每個 `NEXT_PUBLIC_` 變數是否真的可公開
- 非 `NEXT_PUBLIC_` 的變數若在 client component 中引用，值會是 `undefined`——若程式碼靠它做安全判斷，等同判斷失效

---

## 快取（版本差異最大，務必查文件確認）

- `fetch` 與 route 的快取行為在不同版本預設值不同
- 帶有使用者／租戶資料的請求若被靜態化或共用快取 → 跨使用者洩漏
- `unstable_cache`／`revalidateTag` 的 key 與 tag 必須含租戶識別
- 動態渲染的判定（`cookies()`／`headers()` 的使用）會影響是否被快取
- 檢查 `export const dynamic`／`revalidate` 設定與含使用者資料的頁面是否衝突

---

## 中介層限制

- middleware 執行在 edge runtime，能力受限；不要在其中做完整的 DB 權限查詢並假設可靠
- middleware 不應是唯一的授權層——資料存取層仍要驗證

---

## 其他

- `next.config` 的 `images.remotePatterns` 過寬 → 可被當作圖片代理（SSRF 面）
- 正式環境不應輸出 source map（檢查 `productionBrowserSourceMaps`）
- 錯誤頁是否洩漏 stack trace
- `redirect()`／`notFound()` 用於權限不足時，注意不要洩漏資源存在與否
