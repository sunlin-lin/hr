# 法規同步、系統管理與結算邊界

## 系統管理分層

- 系統設定：角色、權限、帳號與系統參數，回答「誰能做什麼」。
- 法規設定：政府資料與公司投保設定，回答「依法怎麼算」。
- 稽核日誌：橫跨薪資、補休、請假、打卡撤銷、核發及權限異動，回答「誰何時改了什麼」。

## 法規四表定案

### `company_regulatory_settings`

**註釋：** 公司職災行業別、投保單位類別及生效歷史。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 主鍵，資料唯一識別碼 |
| `company_id` | `bigint/uuid` | 必填 | 所屬公司外鍵 |
| `occupational_industry_code` | `varchar(30)` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `insurance_unit_type_code` | `varchar(30)` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `effective_from` | `date` | 必填 | 生效開始日 |
| `effective_to` | `date` | 選填 | 生效結束日 |
| `created_by` | `FK` | 必填 | 建立者外鍵 |
| `created_at` | `datetime` | 必填 | 建立時間 |

公司只保存選擇，不複製政府當期費率；同公司有效期間不得重疊。

### `regulatory_dataset_versions`

**註釋：** 平台共用的政府資料歷史版本與原始 Snapshot。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 主鍵，資料唯一識別碼 |
| `dataset_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `version_code` | `varchar(30)` | 必填 | 依原對話之欄位用途；未新增額外規則 |

約束：`UNIQUE(dataset_code, version_code)`；`effective_from` 必填；不用 `is_current`。

### `regulatory_records`

**註釋：** 政府原始資料解析後供 Payroll 查詢的標準化資料。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 主鍵，資料唯一識別碼 |
| `dataset_version_id` | `bigint` | 必填 | 政府法規版本外鍵 |
| `record_key` | `varchar(150)` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `code` | `varchar(100)` | 選填 | 業務代碼 |
| `name` | `varchar(250)` | 選填 | 顯示名稱 |
| `range_from` | `decimal(18,4)` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `range_to` | `decimal(18,4)` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `amount` | `decimal(18,4)` | 選填 | 金額或計算基礎值 |
| `rate` | `decimal(18,8)` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `data` | `json` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `sort_order` | `integer` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `created_at` | `datetime` | 必填 | 建立時間 |

約束：`UNIQUE(dataset_version_id, record_key)`。所得稅特殊結構先放 `data`，暫不另拆表。

### `regulatory_sync_logs`

**註釋：** 每次自動排程或人工同步結果。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 主鍵，資料唯一識別碼 |
| `dataset_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `trigger_type_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |

同步失敗不得破壞既有有效版本。

## 明確不建立／被推翻方案

- `regulatory_sources`
- `regulatory_datasets`
- 永久固定 Resource URL
- Version + Revision
- 政府撤回流程
- 人工核准每個法規版本
- `is_current`

抓取、Metadata、Resource 探索與解析器由程式碼管理；DB 保存版本、原始資料、標準化資料與同步紀錄。

## Payroll 邊界

- 法規模組提供歷史政府資料，Payroll 負責計算。
- 不同法規可使用不同版本，不能只放一個 `regulatory_version_id`。
- 版本依各法規適用基準日選擇，不依系統當天日期。
- 已結算 Payroll 鎖定勞保、健保、勞退、職災與所得稅實際版本；政府後續更新不得改寫。
- 最低工資、加班費等法律公式屬計算邏輯，不因抓到 Open Data 就自動改演算法。

## 稽核日誌

原對話已確認功能包含操作者、時間、操作類型、資料異動前後差異、審核、核發、撤銷、登入登出及權限異動；但尚未找到使用者明確確認的最終表名與逐欄 Schema，因此本文件不自行命名 `audit_logs`。

## 已確認但待後續獨立細化

- 離職生效、未休假、補休與最終薪資結算。
- 報表／統計、通知中心、員工自助入口及附件中心。
- 法定計算公式的版本化實作細節。


