# IFRS 模組｜系統脈絡筆記（帳務底層資料流）

> 用途：在 IFRS 需求訪談前，先掌握 IFRS 計算機所依賴的帳務／財務底層機制。
> 來源：RA003_需求明細表_財務ver8.docx（全文 2,514 行已讀畢）、中嘉新營運系統_新版WBS(SA改這版)
> 建立日期：2026/08/15
> 標註 ⚠ 者為推論或待確認事項，非文件明文。

---

## 一、為什麼要先讀帳務模組

WBS 中 IFRS 模組的「Reference API」區塊列出 IFRS 計算機需要引用的外部資料：

| 來源模組 | 資料 |
| :-- | :-- |
| 帳務模組 | billSummary（帳單彙總）、billData（帳單資料）、billAdj（帳單調整）、giftDisc（贈品折扣）、cycleChrgItem（週期性收費項目）、contrPenalty（合約違約金） |
| 方案設計 | 產品資料 |
| 通用模組 | CommonOptionData（共用代碼）、Company（公司別）、Service（服務別） |
| 拆帳系統 | 拆帳成本 |

這六張帳務資料，全部是由 RA003 描述的機制產生的。**不先搞懂它們怎麼生出來，就無法定義 IFRS 計算機該怎麼讀。**

---

## 二、雙軌資料表：BillData 與 BillData_IFRS

RA003「3.7.10 ACC.POST.REVERSE_反過帳」章節，按鈕說明寫著：

> 「將勾選資料的 **BillData / BillData_IFRS** 狀態恢復為未過帳，並標記傳票序號為已反過帳」

這是整份文件唯一明文提到 `BillData_IFRS` 的地方，但資訊量很大：

1. IFRS 的帳是**獨立一張表**，不是在 BillData 上加欄位
2. 兩張表都有「過帳狀態」，且**同進同退**（一起過帳、一起反過帳）
3. 反過帳章節的查詢結果欄位註明「過帳日期｜來源自 **BillData／IFRS**」，代表兩表結構有相當程度的對應

⚠ **待確認**：兩表是一對一還是一對多？（一筆收款可能拆成多筆 IFRS 認列 → 推測為一對多）誰負責寫入 BillData_IFRS？IFRS 計算機是「即時算」還是「日結批次算」？

---

## 三、主軸一：收款 → 過帳 → 關帳 → 傳票 → ERP

```
收銀機 (ACC.PAY.CASHIER)
   │  現金／線上刷卡(藍新)／匯款／支票
   │  寫入 BillData（實收欄位）
   ▼
繳款過帳 (ACC.POST.EXEC)
   │  勾選帳條 → 過帳 → 產生傳票（暫存）
   │  帳條狀態改為「已過帳」，記錄借方科目／貸方科目
   │  ⚠ 同時更新 BillData_IFRS
   ▼
   ├─→ 反過帳 (ACC.POST.REVERSE)：狀態復原、傳票標記「反過帳」
   │
   ▼
關帳 (ACC.CLOSE.EXEC)
   │  日關帳 → 鎖定該日、拋送發票與傳票至 ERP
   │  月關帳 → 整月鎖定
   │  可重開日關／重開月關（已被月關帳包含的日期不可單獨重開）
   ▼
開立傳票排程 (ACC.VOUCHER.ISSUANCE)  ← 過帳時排入
   │  批次產生傳票，結果回寫 BillData
   ▼
傳票重整列印 (ACC.VOUCHER.REORG) / 傳票維護 (ACC.VOUCHER.MAINTAIN)
   │  傳票分兩型：「日結傳票」與「過帳製票」
   │  重整＝以最新規則修正科目；借貸不平衡則拒絕
   │  維護＝人工新增／修改明細，同樣檢核借貸平衡
   ▼
傳票拋轉至ERP (ACC.VOUCHER.TOERP)   ← 關帳後執行
```

**對 IFRS 的意義**：
- 「關帳」是硬邊界 —— 關帳後不得新增、修改、反過帳。你的日結／攤分作業必須排在關帳**之前**完成。
- 傳票有「借貸平衡檢核」，代表 IFRS 攤分產生的分錄同樣要能通過這道檢核。
- 科目來源是 **ERP**（傳票維護有「重取科目」功能，By 帳管公司同步 ERP 會計科目至 BOSS）。

⚠ **待確認**：IFRS 攤分產生的傳票，算「日結傳票」還是「過帳製票」？還是第三種？

---

## 四、主軸二：開發票 → 上傳財政部 → 拋轉 ERP

```
開立發票排程 (ACC.INV.ISSUANCE)
   │  每日 12:00–13:00
   │  對象：實收日期 = 前一日、且尚未有發票號碼的帳條
   │  方式：「全開全折」
   │    - 開發票：實收總額為正向、收費類型「折扣類」且為負向金額
   │    - 開折讓：實收總額為負向、收費類型「折扣類」且為正向金額
   │  結果回寫 BillData
   │  ★ 具備補跑能力，追溯天數為通用設定
   ▼
上傳財政部電子發票傳輸軟體系統 (ACC.INV.TOMOF)
   │  發票 14:00–15:00、折讓 15:00–16:00
   │  透過 Turnkey 軟體上傳
   │  格式代碼：F0401 開立／F0501 作廢／F0701 註銷／
   │            G0401 折讓開立／G0501 折讓作廢／E0402 空白發票
   ▼
發票拋轉至ERP (ACC.INV.TOERP)
      供營業稅申報用
```

**發票相關的人工作業**（都在 ACC.INV.MAINT 發票資料維護底下）：重開、折讓、作廢、註銷。另有發票號碼維護（年度字軌管理）、彙開發票設定（企業戶多帳戶合併開一張）、中獎發票維護（對獎、簡訊通知、委外列印）、發票明細報表、申告處理作業（4102 預開／4103 異動／4104 補印）。

**對 IFRS 的意義**：這條線走的是**營業稅法規**，跟 IFRS 15 的收入認列是**兩條獨立的線**（詳見會計原則文件 §4-7）。發票開了不等於收入可認，反之亦然。

⚠ **待確認**：發票作廢／折讓／跨期換開時，已認列的 IFRS 收入要不要同步調整？由誰觸發？

---

## 五、時間軸：一天之內的排程順序

| 時間 | 作業 | 備註 |
| :-- | :-- | :-- |
| 00:00 | 銀行代碼主檔同步 (ACC.BANK.SYNC) | 自 FISC 財金資訊官網 |
| 12:00–13:00 | 開立發票排程 (ACC.INV.ISSUANCE) | 處理前一日實收 |
| 14:00–15:00 | 上傳財政部（發票） | Turnkey |
| 15:00–16:00 | 上傳財政部（折讓） | Turnkey |
| 未定 | 退匯更新排程 (ACC.REFUND.UPDATE) | 每日，透過 ERP API |
| 未定 | 開立傳票排程 (ACC.VOUCHER.ISSUANCE) | 過帳時排入 |
| 關帳後 | 傳票拋轉 ERP (ACC.VOUCHER.TOERP) | |
| 發票開立後 | 發票拋轉 ERP (ACC.INV.TOERP) | |

⚠ **IFRS 相關排程（日結、攤分、轉列）要插在這條時間軸的哪裡？** 這是訪談必問 —— 它決定了資料相依性與失敗補救策略。

---

## 六、排程作業的技術規範（已有既定慣例可循）

從 RA006_銀行代碼主檔同步_ver4.xlsx 可以看出這個專案 Batch 作業的標準模式，IFRS 的 Batch（ACC.DAYCLOSE.BATCH）可直接沿用：

- **執行方式**：外部排程平台 Worker 呼叫 `java -jar`，執行時間由平台 Web UI 設定
- **參數外部化**：URL 等組態以 `@Value` 注入 `application.yml`，不寫死在程式
- **Log 機制**：全部輸出 stdout（Spring Boot 預設 Logback ConsoleAppender），Worker 以 `redirectErrorStream=true` 合併截取
- **Log 儲存**：寫入 `SysJobLog`，Summary = stdout 最後一行（max 500 字元），DetailLog = 全文（截頭保尾 max 20,000 字元）
- **成敗判定**：累計 failCount，`> 0` 則 `System.exit(1)` → Worker 判定 FAIL
- **通知**：程式本身不寄信，由 Worker 於 FAIL 時查 `SysUserMain` 取 Owner Email，透過 MsgGW 發送
- **容錯**：per-record 容錯，單筆失敗不中斷整批

---

## 七、後端技術棧（既有規格書已定型，可直接沿用）

從三份既有 RA006 歸納出的架構慣例：

- Spring Boot，`@RestController` / `@Service` / `@Entity`(JPA) / `@ConfigurationProperties` / `@RestControllerAdvice`
- 統一回應包裝 `ManageResponse<T>`：`status` / `code` / `message` / `data` / `source`
- 錯誤代碼 6 位字串，**前 3 碼 = HTTP Status，後 3 碼 = 流水號**（成功 200000、缺必填 400001、參數不合法 400006、外部 API 失敗 500008、系統錯誤 500000）
- `BusinessException` + `ErrorCode` enum + `GlobalExceptionHandler`
- DTO 使用 Java 17 `record`
- API 路徑慣例：`/api/acc/v1/xxx`（財務）、`/billing/v1/xxx`（帳務）
- Repository extends `JpaRepository`，軟刪除以 `isDeleted` 欄位處理

⚠ **待確認**：文件中 Entity 欄位型別使用 `NVARCHAR` / `DATETIME2` / `BIT`（SQL Server 慣用），但 SysJobLog 註明為 **PostgreSQL**。實際 DB 種類需確認。

---

## 八、關鍵人物與窗口（自 WBS 與每日報告整理）

| 角色 | 姓名 | 相關 |
| :-- | :-- | :-- |
| IFRS 資料提供／中嘉審核 | Stanley（林裕庭） | 8/19–8/21 提供 IFRS 計算機資料 |
| IFRS 需求訪談＋詳細設計 | **Frank（林春宏）— 本人** | 8/24 起 |
| IFRS PIC 審核 | Serena（林書伶） | 9/7 起 |
| 財務模組 SA | Aleiku Tsai / 陳建柚 | 已完成銀行代碼、取得銀行清單 RA006，格式可參考 |
| 中嘉窗口 | Mandy（鍾明芳） | 財務模組 Redmine 對口 |
| 帳務模組 SA | Jess Wang / Patricia（蔡思畇） | 帳單產生作業、本期應收維護 |
| 前手／交接方 | 越世 | IFRS 計算機流程曾於 8/5 10:30 與其確認 |

---

## 九、⚠ 最優先確認事項（週一第一件事）

1. **既有的 RA006_IFRS計算機 在哪裡？** WBS 舊版區塊顯示「詳細設計／ACC.IFRS.CALC／珮純／2026-05-04～05-27／100%／預計工時 15」—— 代表五月可能已經有一版完成的規格書。每日報告也提到「比對 RA003、**既有 RA006** 與 PIC 最新格式」。
   → 若屬實，8/24 起的工作是**補完與改格式**，而非從零撰寫，工作性質與難度完全不同。
2. **越世的交接內容與 NextCloud 歷史紀錄** 的存取權限與位置。
3. **BillData_IFRS 的資料字典**（專案字典_v6.12.xlsx 可能有，該檔曾於 Redmine #2637 提供）。
