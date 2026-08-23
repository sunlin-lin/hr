# 組織人事、薪資與人事成本

## 人事定案重點

- `employees` 只代表公司內的人員主檔；不放在職狀態、到離職、部門、職稱、職務或薪資。
- 離職回任建立新的 `employee_employments`。
- 同時間僅一個部門；職務可同時多個；部門、職稱、職務皆留歷史。
- `employment_sequence` 不採用。

## `employees`

**註釋：** 員工個人主檔。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `company_id` | `uuid` | 必填 | 所屬公司外鍵 |
| `employee_code` | `string` | 必填 | 公司內員工編號 |
| `name` | `string` | 必填 | 顯示名稱 |
| `gender` | `string` | 必填 | 性別代碼 |
| `identity_number_encrypted` | `binary` | 必填 | 身分證加密值 |
| `identity_number_hash` | `binary` | 必填 | 身分證查詢 Hash |
| `birthday_encrypted` | `binary` | 必填 | 出生年月日加密值 |
| `phone_encrypted` | `binary` | 必填 | 電話加密值 |
| `email_encrypted` | `binary` | 必填 | Email 加密值 |
| `address_encrypted` | `binary` | 必填 | 地址加密值 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |
| `deleted_at` | `datetime` | 選填 | Soft Delete 時間 |

**明確不含：** `status`、`hire_date`、`leave_date`。

## `employee_employments`

**註釋：** 員工每次任職關係；回任新增一筆。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employee_id` | `uuid` | 必填 | 員工外鍵 |
| `employment_type_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |

## 人事歷史表

### `employee_department_histories`

**註釋：** 任職期間的部門歸屬歷史。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employment_id` | `uuid` | 必填 | 任職紀錄外鍵 |
| `department_id` | `uuid` | 必填 | 部門外鍵 |
| `effective_from` | `date` | 必填 | 生效開始日 |
| `effective_to` | `date` | 選填 | 生效結束日 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |

約束：同一任職在同一時間只能有一筆有效部門，期間不可重疊。

### `job_titles`

**註釋：** 系統預設及公司自訂職稱。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `company_id` | `uuid` | 選填 | 所屬公司外鍵 |

### `employee_job_title_histories`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employment_id` | `uuid` | 必填 | 任職紀錄外鍵 |
| `job_title_id` | `uuid` | 必填 | 職稱外鍵 |
| `effective_from` | `date` | 必填 | 生效開始日 |
| `effective_to` | `date` | 選填 | 生效結束日 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |

### `job_positions`

**註釋：** 職務主檔；職務與職稱分離。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `company_id` | `uuid` | 選填 | 所屬公司外鍵 |
| `code` | `string` | 必填 | 業務代碼 |
| `name` | `string` | 必填 | 顯示名稱 |
| `description` | `string` | 必填 | 用途或異動說明 |
| `is_system` | `boolean` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `status` | `string` | 必填 | 狀態代碼，不使用 DB ENUM |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |
| `deleted_at` | `datetime` | 選填 | Soft Delete 時間 |

### `employee_job_position_histories`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employment_id` | `uuid` | 必填 | 任職紀錄外鍵 |
| `job_position_id` | `uuid` | 必填 | 職務外鍵 |
| `effective_from` | `date` | 必填 | 生效開始日 |
| `effective_to` | `date` | 選填 | 生效結束日 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |

## `employee_dependents`

**註釋：** 薪資扣繳／報稅所需扶養親屬及資格條件。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employee_id` | `uuid` | 必填 | 員工外鍵 |
| `name` | `string` | 必填 | 顯示名稱 |
| `identity_number_encrypted` | `binary` | 必填 | 身分證加密值 |
| `identity_number_hash` | `binary` | 必填 | 身分證查詢 Hash |
| `birthday_encrypted` | `binary` | 必填 | 出生年月日加密值 |
| `relationship_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `is_student` | `boolean` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `is_disabled` | `boolean` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `is_unable_to_work` | `boolean` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `is_cohabiting` | `boolean` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `effective_date` | `date` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `end_date` | `date` | 選填 | 結束日期 |
| `status` | `string` | 必填 | 狀態代碼，不使用 DB ENUM |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |
| `deleted_at` | `datetime` | 選填 | Soft Delete 時間 |

親屬代碼直接寫欄位註釋，不另開表：1配偶、2父親、3母親、4子女、5兄弟姊妹、6祖父母、7孫子女、8其他。

## `employee_withholding_settings`

**註釋：** 每月薪資扣繳方式及有效期間。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint/uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employee_id` | `FK` | 必填 | 員工外鍵 |
| `withholding_method_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |

## 薪資核心

### `salary_items`

**註釋：** 系統預設或公司自訂薪資項目。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `company_id` | `uuid` | 選填 | 所屬公司外鍵 |
| `code` | `string` | 必填 | 業務代碼 |
| `name` | `string` | 必填 | 顯示名稱 |
| `type_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |

### `employee_salary_settings`

**註釋：** 任職的長期薪資項目設定與調薪歷史。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employment_id` | `uuid` | 必填 | 任職紀錄外鍵 |
| `salary_item_id` | `uuid` | 必填 | 薪資項目外鍵 |
| `calculation_type_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `amount` | `decimal` | 必填 | 金額或計算基礎值 |
| `start_date` | `date` | 必填 | 開始日期 |
| `end_date` | `date` | 選填 | 結束日期 |
| `description` | `string` | 必填 | 用途或異動說明 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |
| `deleted_at` | `datetime` | 選填 | Soft Delete 時間 |

### `payroll_settings`

**註釋：** 公司計薪週期與發薪制度。已確認存在；原對話完整欄位未在最終摘要全部重列，實作前按原文補錄。

### `payroll_periods`

**註釋：** 實際計薪期間，如 26 日至次月 25 日；包含期間開始、結束與發薪日。

### `payrolls`

**註釋：** 員工某計薪期薪資單／結算主檔；保存應發、扣款、實發與鎖定狀態。

### `payroll_details`

**註釋：** 當期實際薪資明細，可來自長期設定、系統計算、臨時新增或人工調整。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `原對話此處未重列` | 待核對 | 主鍵，資料唯一識別碼 |
| `payroll_id` | `原對話此處未重列` | 待核對 | 薪資結算外鍵 |
| `salary_item_id` | `nullable` | 選填 | 薪資項目外鍵 |
| `item_name` | `原對話此處未重列` | 待核對 | 依原對話之欄位用途；未新增額外規則 |
| `type_code` | `原對話此處未重列` | 待核對 | 依原對話之欄位用途；未新增額外規則 |
| `source_type_code` | `原對話此處未重列` | 待核對 | 依原對話之欄位用途；未新增額外規則 |

### `employee_salary_bank_accounts`

**註釋：** 員工發薪銀行帳戶；敏感帳號需加密。

### `payroll_payments`

**註釋：** 實際薪資發放方式、時間、金額與結果。

## 人事成本

### `personnel_cost_items`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `company_id` | `uuid` | 選填 | 所屬公司外鍵 |
| `code` | `string` | 必填 | 業務代碼 |
| `name` | `string` | 必填 | 顯示名稱 |
| `type_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `is_active` | `boolean` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `description` | `string` | 必填 | 用途或異動說明 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |
| `deleted_at` | `datetime` | 選填 | Soft Delete 時間 |

### `personnel_costs`

**註釋：** 某員工在某成本期間的實際公司負擔。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `company_id` | `uuid` | 必填 | 所屬公司外鍵 |
| `employee_id` | `uuid` | 必填 | 員工外鍵 |
| `employment_id` | `uuid` | 必填 | 任職紀錄外鍵 |
| `personnel_cost_item_id` | `uuid` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `payroll_id` | `uuid` | 選填 | 薪資結算外鍵 |
| `cost_date` | `date` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `cost_period` | `string` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `amount` | `decimal` | 必填 | 金額或計算基礎值 |
| `source_type_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `description` | `string` | 必填 | 用途或異動說明 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |


