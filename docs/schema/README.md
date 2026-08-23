# HR 系統 Schema 規劃

> 本文件只整理目前討論中已經明確定義／確認的 Schema。尚未定稿的模組不自行補表，避免把推測寫成規格。

## 已確認 Schema 清單

### 員工／人事

- `employee_dependents` — 員工配偶／扶養親屬與列報扶養的生效歷史。
- `employee_withholding_settings` — 員工每月薪資所得扣繳方式與生效歷史。

`employee_withholding_settings.withholding_method_code`：

- `1` = 依薪資所得扣繳稅額表
- `2` = 按全月給付總額 5%

經常性／非經常性薪資不是員工屬性，應由薪資項目的給付性質決定。

### 排班

- `employee_schedules` — 員工實際班表；作為工作日／休息日／假日、出勤、請假及加班資格判斷的基礎。已產生的歷史班表不因後續排班規則改變而被覆蓋。

### 政府法規資料同步／法規設定

核心 4 張：

- `company_regulatory_settings`
- `regulatory_dataset_versions`
- `regulatory_records`
- `regulatory_sync_logs`

不建立：

- `regulatory_sources`
- `regulatory_datasets`

政府來源、API、Adapter、Dataset 探索及解析邏輯固定由程式碼管理；資料庫不保存永久固定的 Resource URL。

---

## `company_regulatory_settings`

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

---

## `regulatory_dataset_versions`

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

---

## `regulatory_records`

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

---

## `regulatory_sync_logs`

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

---

## 法規同步核心規則

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

## 尚未在本文件硬寫完整欄位的既有模組

我們已討論並確定需求邏輯，但目前可取得的定案內容不足以安全重建完整欄位，因此暫不自行補 Schema：

- 公司／組織／部門／職稱／職務／聘僱性質
- 完整員工主檔與任職歷史
- 班別／排班體系／循環班／零工排班
- 打卡、撤銷打卡、GPS、補登申請
- 加班申請與審核
- 補休核發、使用、到期轉薪資、撤銷／重新核發
- 請假制度、假別、額度、公司贈與特休／補休
- 完整薪資設定、薪資項目、調薪、Payroll 結算
- 角色／權限／稽核日誌

後續每個模組確認欄位後，再逐步補入此目錄，避免把未定案內容誤當正式 Schema。