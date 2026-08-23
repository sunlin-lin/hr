> [!WARNING]
> 本文件早期段落含已被後續對話更正的名稱與不完整恢復內容。開發前請先閱讀 [定案恢復總索引](./recovered/00-INDEX.md)；其中的可信度分級與衝突說明優先於本文件舊段落。

# HR 系統 Schema 規劃

> 本文件整理目前討論中已經明確定義或已確認用途的 Schema。欄位已完整定案者直接列欄位；其餘先列出已經討論確認的表與責任，不自行新增未確認欄位。

## 目前已討論／定義的 Schema 總表

### 組織／人事

- `employees` — 員工基本資料；`status` 已確認不放在主表。
- `employee_employments` — 到職／離職與重新任職歷史。
- `departments` — 部門樹狀階層，例如總經理室 → 研發部 → 研發一處。
- `employee_departments` — 員工部門歸屬歷史；員工任一時間點只能有一個所屬部門，但可因異動形成歷史。
- `job_positions` — 職稱主檔，可預先建立常見職稱。
- `employee_positions` — 員工職務；同一員工可有多個職務。
- `employee_dependents` — 員工配偶／扶養親屬、親屬關係代碼與列報扶養生效歷史。
- `employee_withholding_settings` — 員工每月薪資所得扣繳方式與生效歷史。

`employee_withholding_settings.withholding_method_code`：

- `1` = 依薪資所得扣繳稅額表
- `2` = 按全月給付總額 5%

經常性／非經常性薪資不是員工屬性，由薪資項目的給付性質決定。

---

### 班表／排班

- `employee_schedules` — 員工實際班表；作為工作日、休息日、假日、出勤、請假、加班資格判斷的基礎。歷史班表不因後續排班規則改變而被覆蓋。

已確認需支援：固定週班、循環班（例如做二休二）、輪班、零工／彈性排班、假日班、中途調班、請假後仍保留原班表事實。

---

### 加班

- `overtime_requests` — 員工加班申請。
- `overtime_approvals` — 加班審核／拒絕／撤回等歷史。
- `overtime_compensations` — 已核准加班最終採用的補償方式。

核心規則：

- 一筆加班不可拆分為部分加班費＋部分補休。
- 補償方式為 `1=加班費`、`2=補休`。
- 實際打卡超過申請時段，不自動增加認列時間；以申請時段為準。
- 支援跨日加班。
- 拒絕後可重新申請，歷史保留。

---

### 補休

- `compensatory_leave_credits` — 每一批補休額度。
- `compensatory_leave_rate_snapshots` — 補休核發當下所選薪資計價基準 Snapshot。
- `compensatory_leave_transactions` — 補休取得、預約、使用、返還、到期轉薪、撤銷等帳本。
- `compensatory_leave_allocations` — 補休實際從哪些批次被使用／凍結／返還。

核心規則：

- 補休來源可為加班、公司贈與、人工調整。
- 加班轉補休時由有權限核發者選擇公司允許的計價基準。
- 核發後 Snapshot 不受後續調薪影響。
- 補休可部分使用。
- 使用採最早到期優先。
- 尚未核准的使用申請需凍結額度。
- 取消後原路返還到原補休批次。
- 到期當日仍可使用；到期未使用部分轉薪資。
- 核發後可撤銷並重新核發，但不修改原始 Snapshot。

---

### 請假／假別

- `leave_types` — 假別定義。
- `leave_type_rules` — 假別法規／公司規則與有效期間。
- `leave_entitlements` — 員工實際取得的假別額度批次。
- `leave_balances` — 假別目前餘額快取／彙總。
- `leave_balance_transactions` — 假別額度完整異動帳本。
- `leave_requests` — 請假申請主單。
- `leave_request_details` — 請假日期／時段／假別明細，可支援一張請假單混合多種假別。
- `leave_request_approvals` — 請假審核歷史。
- `leave_request_allocations` — 每筆請假明細實際扣到哪一批假別額度。
- `leave_request_documents` — 診斷證明、死亡證明、親屬關係證明等附件／驗證資料。
- `leave_events` — 結婚、親屬死亡、生產、流產、懷孕、配偶分娩、職災等特殊事件來源。

核心規則：

- 特休、福利假、補休是不同假別。
- 假別、額度、請假、薪資分離。
- 原始核發額度不可用 UPDATE 抹掉歷史。
- 請假可混合不同假別。
- 實際扣哪一批額度必須透過 allocation 保留。
- 到期／離職後不可再使用，但額度與歷史保留。

---

### 公司贈與假別

- `company_leave_grant_batches` — 公司贈與批次。
- `company_leave_grants` — 批次中每位員工的實際贈與紀錄。

核心規則：

- 公司可贈與特休、補休、福利假等允許公司核發的假別。
- 員工不能互相轉贈。
- `pay_type_code` 僅：`1=有薪`、`2=無薪`。
- 一個批次不可混用假別、薪資類型、額度或有效期間。
- `granted_minutes > 0`。
- `effective_from`、`effective_to` 皆必填，且核發後不可直接修改。
- 公司贈與直接核發，不走員工請假申請。
- 批次逐員工處理，失敗員工可單獨重新核發，不整批 rollback。
- 生效前可撤銷；已使用歷史不能被撤銷抹除。
- 到期或離職後不可使用，但資料保留。

---

### 薪資／Payroll

- `payroll_pending_items` — 尚未進薪資週期，但之後需要結算的項目，例如補休到期轉薪。
- `payroll_settlements` — 員工薪資週期結算及鎖定狀態。

已確認薪資設計原則：

- 員工薪資與員工主檔分離，保留調薪生效歷史。
- 薪資項目與員工薪資設定分離。
- 發薪時可臨時增加一次性發放／扣除項目。
- 薪資採事後結算；事件發生時保存依據，薪資週期再計算。
- 結算後不可直接異動，只能後續補發、扣回、更正。
- 政府法規版本與其他計算基準在結算時需留下 Snapshot。

---

## 政府法規資料同步／法規設定

核心 4 張：

- `company_regulatory_settings`
- `regulatory_dataset_versions`
- `regulatory_records`
- `regulatory_sync_logs`

明確不建立：

- `regulatory_sources`
- `regulatory_datasets`

政府來源、API、Adapter、Dataset 探索及解析邏輯固定由程式碼管理；資料庫不保存永久固定的 Resource URL。

### `company_regulatory_settings`

用途：保存公司自己的投保相關設定及其生效歷史。

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | BIGINT | 必填 | 主鍵 |
| `company_id` | BIGINT | 必填 | 公司 ID |
| `occupational_industry_code` | VARCHAR(30) | 必填 | 職災行業別代碼 |
| `insurance_unit_type_code` | VARCHAR(30) | 必填 | 投保單位類別代碼 |
| `effective_from` | DATE | 必填 | 生效日期 |
| `effective_to` | DATE | 選填 | 結束日期；目前有效可為 NULL |
| `created_by` | BIGINT | 必填 | 設定人 |
| `created_at` | DATETIME | 必填 | 建立時間 |

設計原則：只保存公司的選擇，不把政府當期職災費率複製進公司設定；實際費率依適用日期由政府法規版本取得。

### `regulatory_dataset_versions`

用途：保存每一類政府結構化資料的歷史版本與政府原始資料 Snapshot。

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | BIGINT | 必填 | 法規版本 ID |
| `dataset_code` | INT | 必填 | 程式固定的政府資料類型代碼 |
| `version_code` | VARCHAR(30) | 必填 | 統一轉為西元版本，例如 `2026-01` |
| `effective_from` | DATE | 必填 | 適用開始日 |
| `effective_to` | DATE | 選填 | 適用結束日；可由下一版本生效日前一天補齊 |
| `government_resource_id` | VARCHAR(150) | 選填 | 當次政府 Resource ID，供追蹤 |
| `source_modified_at` | DATETIME | 選填 | 政府資料修改時間 |
| `synced_at` | DATETIME | 必填 | 系統抓取時間 |
| `checksum` | VARCHAR(128) | 必填 | 原始內容 Hash |
| `record_count` | INT | 選填 | 資料筆數 |
| `raw_format_code` | INT | 必填 | 原始格式，例如 JSON／CSV／XML |
| `raw_data` | LONGTEXT | 必填 | 政府原始資料完整 Snapshot |
| `created_at` | DATETIME | 必填 | 建立時間 |

建議約束：`UNIQUE(dataset_code, version_code)`。

目前程式支援的政府結構化資料類型：

1. 勞保投保薪資級距
2. 勞就保保費分擔
3. 健保投保金額級距
4. 勞退月提繳級距
5. 職災投保薪資級距
6. 職災行業別／費率
7. 薪資所得扣繳稅額表

### `regulatory_records`

用途：把政府原始資料解析成 Payroll 可直接查詢的標準化計算資料。Payroll 不直接解析 `raw_data`。

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | BIGINT | 必填 | 主鍵 |
| `dataset_version_id` | BIGINT | 必填 | 所屬法規版本 |
| `record_key` | VARCHAR(150) | 必填 | 該版本內唯一識別 |
| `code` | VARCHAR(100) | 選填 | 等級、行業或政府代碼 |
| `name` | VARCHAR(250) | 選填 | 名稱 |
| `range_from` | DECIMAL(18,4) | 選填 | 級距起點 |
| `range_to` | DECIMAL(18,4) | 選填 | 級距終點 |
| `amount` | DECIMAL(18,4) | 選填 | 投保薪資、扣繳額等金額 |
| `rate` | DECIMAL(18,8) | 選填 | 費率 |
| `data` | JSON | 必填 | 特殊的標準化欄位 |
| `sort_order` | INT | 選填 | 排序 |
| `created_at` | DATETIME | 必填 | 建立時間 |

關聯：`dataset_version_id -> regulatory_dataset_versions.id`。

建議約束：`UNIQUE(dataset_version_id, record_key)`。

所得稅目前不另拆專表，特殊結構先放 `data`；若實際串接後確認通用結構不適合，再獨立調整。

### `regulatory_sync_logs`

用途：保存政府資料每次自動排程或人工觸發的同步結果。

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | BIGINT | 必填 | 同步紀錄 ID |
| `dataset_code` | INT | 必填 | 本次同步的政府資料類型 |
| `trigger_type_code` | INT | 必填 | `1=自動排程`、`2=人工觸發` |
| `started_at` | DATETIME | 必填 | 開始時間 |
| `finished_at` | DATETIME | 選填 | 結束時間 |
| `status_code` | INT | 必填 | `1=執行中`、`2=更新成功`、`3=失敗`、`4=無異動` |
| `dataset_version_id` | BIGINT | 選填 | 若建立新版本則指向該版本 |
| `government_resource_id` | VARCHAR(150) | 選填 | 當次取得的政府 Resource |
| `records_received` | INT | 選填 | 政府回傳筆數 |
| `error_message` | TEXT | 選填 | 失敗原因 |
| `created_at` | DATETIME | 必填 | 建立時間 |

同步失敗時不更新法規資料，Payroll 繼續使用既有有效版本。

### 法規同步核心規則

1. 政府 API／Adapter／Dataset 抓取方式由程式碼固定管理。
2. 不寫死永久 Resource URL；同步時取得當下實際 Resource。
3. 政府正式資料自動抓取、解析版本／生效日、驗證、保存。
4. `version_code` 統一使用西元格式，例如 `2026-01`。
5. `effective_from` 必填；`effective_to` 可空。
6. 新版本建立後，可自動將前一版 `effective_to` 補為新版本 `effective_from - 1 day`。
7. 原始資料以 `raw_format_code + LONGTEXT raw_data` 保存，支援 JSON／CSV／XML。
8. 同步異常不取代既有有效版本。
9. Payroll 依計算所需的適用日期取得正確版本。
10. Payroll 結算時必須保存實際使用的勞保、健保、勞退、職災、所得稅法規版本；後續政府更新不得改變已結算結果。

---

## 尚需從先前對話補回完整欄位的已討論 Schema

以下不是「尚未討論」，而是已經討論過；目前此文件尚缺完整欄位細節，後續應按先前定案內容逐表補齊，不重新發明規格：

- 角色／權限相關 Schema
- 完整 `employees` 欄位
- `employee_employments`
- `departments`
- `employee_departments`
- `job_positions`
- `employee_positions`
- 完整排班模板／循環班 Schema
- 打卡、撤銷打卡、GPS、補登申請 Schema
- 完整薪資設定、薪資項目、調薪歷史、人事成本 Schema
- 稽核日誌 Schema

這些會以既有討論為準補錄，不把新猜測混入已定案規格。