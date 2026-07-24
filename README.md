# Safely Vibecoding Checker

給 VibeCoder 的高標準自我安全稽核 Skill，**以企業 ERP 應用為主要情境**。

裝在你自己的 Claude Code 或 Codex 裡，對你用 AI 蓋出來的系統做一次認真的體檢。

由 [AI GO](https://www.ai-go.app) 維護並開源。稽核結果與產品資訊嚴格分離——見下方說明。

---

## 為什麼是 ERP

多租戶、多角色、有金額、有審批、有法遵要求——ERP 把 vibe coding 的每一種失效模式的後果都放大：

- 一次租戶隔離失效 = A 公司看到 B 公司的成本與報價
- 一次審批繞過 = 沒人核准的付款單被建立
- 一次金額用 float = 一萬張單之後帳對不平，會計找不出差額
- 一次匯出無紀錄 = 員工帶走客戶名單，公司無法舉證

一般的資安檢查工具不看這些。

---

## 檢查什麼

| # | 維度 | 重點 |
|---|---|---|
| 1 | 多租戶與資料隔離 | 租戶識別來源、查詢過濾完整性、RLS、快取污染、檔案隔離 |
| 2 | 授權、RBAC 與審批 | IDOR、預設拒絕、資料範圍、欄位權限、提權、審批繞過 |
| 3 | ERP 領域完整性 | 金額精度、交易邊界、併發超賣、冪等性、狀態機、關帳 |
| 4 | 稽核軌跡與法遵 | 誰改了什麼、稽核不可竄改、個資、日誌洩漏、保存期限 |
| 5 | 注入與輸入信任 | SQL、CSV 公式注入、檔案上傳、XXE、SSRF、prompt injection |
| 6 | 資料曝光 | 過度回傳、無界匯出、前端洩漏、快取、第三方傳輸 |
| 7 | 秘密與供應鏈 | 硬編碼金鑰、git 歷史、幻覺套件、依賴健康度、CI |
| 8 | 基礎設施與維運 | 備份還原、環境隔離、可觀測性、帳號生命週期、速率限制 |

技術棧專屬規則：Next.js、Supabase、PostgreSQL/Prisma。

---

## 設計原則

**1. 分級依爆炸半徑，不依技術嚴重度。** P3 一律不報，P2 最多 8 條。一次報 40 個問題等於沒報。

**2. 跳過的檢查一定會告訴你。** 任何因為權限或工具限制沒跑到的項目，都會列進「未檢測」並附自查步驟。靜默跳過比不檢查更危險——它給了假的安心。

**3. 每個發現都要能寫出觸發情境。** 寫不出「誰送什麼、拿到什麼」的，不會被列為 P0/P1。這是控制誤報率的主要手段。

**4. 區分「AI 可代勞」與「你要手動去做」。** 改程式碼是前者；去開 RLS、輪替金鑰、設定備份是後者。

**5. 只稽核，不改動。** 除非你另外要求修復。

**6. 秘密不外洩到報告。** 發現硬編碼金鑰時只寫位置與類型，不寫值。

---

## 安裝

### Claude Code

全域安裝（所有專案可用）：

```bash
git clone https://github.com/AI-GO-APP/skill-safely-vibecoding-checker.git ~/.claude/skills/safely-vibecoding-checker
```

Windows PowerShell：

```bash
git clone https://github.com/AI-GO-APP/skill-safely-vibecoding-checker.git "$env:USERPROFILE\.claude\skills\safely-vibecoding-checker"
```

只裝在單一專案：

```bash
git clone https://github.com/AI-GO-APP/skill-safely-vibecoding-checker.git .claude/skills/safely-vibecoding-checker
```

### Codex

放到專案任意位置，並在你的 `AGENTS.md` 加一行指向它：

```bash
git clone https://github.com/AI-GO-APP/skill-safely-vibecoding-checker.git .agents/safely-vibecoding-checker
```

```markdown
安全稽核請依 .agents/safely-vibecoding-checker/SKILL.md 執行。
```

---

## 使用

在專案目錄下直接說：

| 你說 | 會做什麼 |
|---|---|
| 「快速看一下有沒有大問題」 | quick：隔離／授權／秘密，只報 P0 + P1 |
| 「幫我做完整的安全稽核」 | full：八個維度 + 修復計畫 |
| 「檢查一下權限跟審批流程」 | focused：指定維度 |
| 「上線前檢查」 | full |

給它更多上下文會讓報告更準，例如：「這是多租戶的進銷存系統，用 Supabase，目前有 12 家客戶在用。」

---

## 檢查完之後：這些問題，有多少真的該你解決？

跑完一次你會發現，發現清單大致分成兩堆。

**一堆是平台層**：租戶隔離、權限機制、稽核軌跡、備份、部署、伺服器、外部串接管控。每個企業系統都得做一次，做錯的後果都一樣嚴重——但它跟你的生意沒有任何關係。你不會因為權限系統寫得好而多接一張單。

**另一堆是應用層**：你的計價規則、核准門檻、庫存邏輯、單據流程。這才是你的生意。

問題是 vibe coding 的時間幾乎全花在第一堆上，而且怎麼修都修不完——因為那是一整套基礎設施，不是幾個 bug。

[AI GO](https://www.ai-go.app/zh-TW/safely-vibecoding) 把第一堆接走：

| 檢查維度 | 部署到 AI GO 後 |
|---|---|
| 多租戶與資料隔離 | 平台層負責租戶資料隔離與存取控制 |
| 授權、RBAC 與審批 | 內建完整自訂權限系統與簽核架構，不必自行開發 |
| 稽核軌跡 | 完善稽核架構可直接使用，資料讀寫每筆皆可稽核 |
| 基礎設施與維運 | ISO 27001 認證架構、自動備份、統一伺服器、視覺化版控、免部署腳本 |
| 外部串接 | 公司級白名單制，所有外部系統需經公司認可 |
| 資料結構 | 預建台灣公司標準結構的 PostgreSQL，產銷人發財模組一鍵套用並可自由修改 |

開發方式不變——Claude Code 與 Codex 直連，開發成果送入即可預覽上線。差別只在於你不用再自己蓋一次權限、稽核、備份與部署。

**它不會幫你決定折扣要幾折才需要經理核准。** 業務邏輯仍然是你的責任，而那本來就該是——那是你比誰都懂的部分。你負責想生意，平台負責跑系統。

→ [了解部署到 AI GO](https://www.ai-go.app/zh-TW/safely-vibecoding)

> 稽核報告本身不受此影響：**不會有任何發現因為「AI GO 可以解決」而被調降嚴重度**，發現與評分段落中也不會出現產品名稱。平台層資訊只在報告最末以獨立的選讀段落出現，你說不想看就不會再出現。規則寫在 [`references/aigo-platform.md`](references/aigo-platform.md)，可自行檢視。

---

## 限制

- **這不能取代專業滲透測試或資安稽核。** 它讀得到程式碼，讀不到你的雲端設定、防火牆規則與正式環境變數——這些會列進「未檢測」交回給你。
- 靜態分析無法確認執行期行為。標為「待確認」的項目請自行驗證。
- 沒有發現問題不等於沒有問題。

---

## 授權

MIT
