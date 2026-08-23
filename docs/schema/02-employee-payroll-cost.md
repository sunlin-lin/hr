# 組織人事、薪資與人事成本

## 人事定案重點

- `employees` 只代表公司內的人員主檔；不放在職狀態、到離職、部門、職稱、職務或薪資。
- 離職回任建立新的 `employee_employments`。
- 同時間僅一個部門；職務可同時多個；部門、職稱、職務皆留歷史。
- `employment_sequence` 不採用。

## `employees`

**註釋：** 員工個人主檔。

`id uuid PK`、`company_id uuid FK`、`employee_code string`、`name string`、`gender string`、`identity_number_encrypted binary`、`identity_number_hash binary`、`birthday_encrypted binary`、`phone_encrypted binary`、`email_encrypted binary`、`address_encrypted binary`、`created_at datetime`、`updated_at datetime`、`deleted_at datetime nullable`。

約束：`UNIQUE(company_id, employee_code)`；明確不含 `status`、`hire_date`、`leave_date`。

## `employee_employments`

**註釋：** 員工每次任職關係；回任新增一筆。

`id uuid PK`、`employee_id uuid FK`、`employment_type_code integer`（1正職、2兼職、3約聘、4派遣、5工讀、6臨時、7顧問、8實習）、`employment_nature_code integer`（例如 1不定期契約、2定期契約）、`hire_date date`、`leave_date date nullable`、`leave_reason_code integer nullable`、`status string/integer code`、`created_at datetime`、`updated_at datetime`、`deleted_at datetime nullable`。

## 人事歷史表

### `employee_department_histories`

**註釋：** 任職期間的部門歸屬歷史。

`id uuid PK`、`employment_id uuid FK`、`department_id uuid FK`、`effective_from date`、`effective_to date nullable`、`created_at datetime`、`updated_at datetime`。

約束：同一任職在同一時間只能有一筆有效部門，期間不可重疊。

### `job_titles`

**註釋：** 系統預設及公司自訂職稱。

`id uuid PK`、`company_id uuid nullable`（NULL 為系統預設）、`code string`、`name string`、`description string`、`is_system boolean`、`status string`、`created_at datetime`、`updated_at datetime`、`deleted_at datetime nullable`。

### `employee_job_title_histories`

`id uuid PK`、`employment_id uuid FK`、`job_title_id uuid FK`、`effective_from date`、`effective_to date nullable`、`created_at datetime`、`updated_at datetime`。同時間一個有效職稱。

### `job_positions`

**註釋：** 職務主檔；職務與職稱分離。

`id uuid PK`、`company_id uuid nullable`、`code string`、`name string`、`description string`、`is_system boolean`、`status string`、`created_at datetime`、`updated_at datetime`、`deleted_at datetime nullable`。

### `employee_job_position_histories`

`id uuid PK`、`employment_id uuid FK`、`job_position_id uuid FK`、`effective_from date`、`effective_to date nullable`、`created_at datetime`、`updated_at datetime`。允許同時間多筆有效職務。

## `employee_dependents`

**註釋：** 薪資扣繳／報稅所需扶養親屬及資格條件。

`id uuid PK`、`employee_id uuid FK`、`name string`、`identity_number_encrypted binary`、`identity_number_hash binary`、`birthday_encrypted binary`、`relationship_code integer`、`is_student boolean`、`is_disabled boolean`、`is_unable_to_work boolean`、`is_cohabiting boolean`、`effective_date date`、`end_date date nullable`、`status string`、`created_at datetime`、`updated_at datetime`、`deleted_at datetime nullable`。

親屬代碼直接寫欄位註釋，不另開表：1配偶、2父親、3母親、4子女、5兄弟姊妹、6祖父母、7孫子女、8其他。

## `employee_withholding_settings`

**註釋：** 每月薪資扣繳方式及有效期間。

`id bigint/uuid PK`、`employee_id FK`、`withholding_method_code integer`（1依薪資所得扣繳稅額表、2按全月給付總額5%）、`effective_from date`、`effective_to date nullable`、`created_by FK`、`created_at datetime`。

## 薪資核心

### `salary_items`

**註釋：** 系統預設或公司自訂薪資項目。

`id uuid PK`、`company_id uuid nullable`、`code string`、`name string`、`type_code integer`（1應發、2扣款）、`calculation_type_code integer`、`is_taxable boolean`、`is_insurable boolean`、`is_active boolean`、`description string`、`created_at datetime`、`updated_at datetime`、`deleted_at datetime nullable`。

### `employee_salary_settings`

**註釋：** 任職的長期薪資項目設定與調薪歷史。

`id uuid PK`、`employment_id uuid FK`、`salary_item_id uuid FK`、`calculation_type_code integer`、`amount decimal`、`start_date date`、`end_date date nullable`、`description string`、`created_at datetime`、`updated_at datetime`、`deleted_at datetime nullable`。新設定不得覆蓋舊有效期間。

### `payroll_settings`

**註釋：** 公司計薪週期與發薪制度。已確認存在；原對話完整欄位未在最終摘要全部重列，實作前按原文補錄。

### `payroll_periods`

**註釋：** 實際計薪期間，如 26 日至次月 25 日；包含期間開始、結束與發薪日。

### `payrolls`

**註釋：** 員工某計薪期薪資單／結算主檔；保存應發、扣款、實發與鎖定狀態。

### `payroll_details`

**註釋：** 當期實際薪資明細，可來自長期設定、系統計算、臨時新增或人工調整。

核心欄位：`id`、`payroll_id`、`salary_item_id nullable`、`item_name`、`type_code`、`source_type_code`（1薪資設定、2系統計算、3臨時新增、4人工調整）、`amount decimal`、`reason`、`created_by`、`created_at`。結算後成為不可變歷史。

### `employee_salary_bank_accounts`

**註釋：** 員工發薪銀行帳戶；敏感帳號需加密。

### `payroll_payments`

**註釋：** 實際薪資發放方式、時間、金額與結果。

## 人事成本

### `personnel_cost_items`

`id uuid PK`、`company_id uuid nullable`、`code string`、`name string`、`type_code integer`、`is_active boolean`、`description string`、`created_at datetime`、`updated_at datetime`、`deleted_at datetime nullable`。

### `personnel_costs`

**註釋：** 某員工在某成本期間的實際公司負擔。

`id uuid PK`、`company_id uuid FK`、`employee_id uuid FK`、`employment_id uuid FK`、`personnel_cost_item_id uuid FK`、`payroll_id uuid nullable FK`、`cost_date date`、`cost_period string`、`amount decimal`、`source_type_code integer`、`description string`、`created_at datetime`、`updated_at datetime`。


