# 公司、權限、組織、員工

## 1. `companies` ✅

**用途：** SaaS Tenant／公司或個人型主體。

**定案欄位：**

| 欄位 | 型態方向 | 必要 | 說明 |
|---|---|---:|---|
| id | UUID | 是 | PK |
| company_code | STRING | 是 | 全域唯一公司代碼 |
| company_type | STRING | 是 | 公司／個人等類型 |
| legal_type | STRING | 是 | 法律型態 |
| tax_id | STRING | 視類型 | 統編等識別 |
| name | STRING | 是 | 正式名稱 |
| short_name | STRING | 否 | 簡稱 |
| status | STRING | 是 | 公司狀態 |
| registered_postal_code/city/district/address | STRING | 否 | 登記地址 |
| actual_postal_code/city/district/address | STRING | 否 | 實際地址 |
| invoice_postal_code/city/district/address | STRING | 否 | 發票地址 |
| created_at/updated_at | DATETIME | 是 | 建立／更新 |
| deleted_at | DATETIME | 否 | Soft Delete |

**定案：** 地址直接放本表，不拆 address 表；公司代碼與 PK 分離。公司型代碼為統編加三碼只增不減流水號；個人型為 YYYYMMDD 加三碼流水號；不使用分隔符。

## 2. `company_contacts` ✅

**用途：** 公司負責人、業務、帳務聯絡資料。

| 欄位 | 說明 |
|---|---|
| id | PK |
| company_id | FK → companies |
| contact_type | OWNER／SALES／ACCOUNTING 等一般代碼 |
| name | 姓名 |
| identity / identity_hash | 加密值／查詢 Hash |
| birthday | 加密 |
| phone / phone_hash | 加密值／查詢 Hash |
| email / email_hash | 加密值／查詢 Hash |
| created_at/updated_at | 時間欄位 |

關係：`companies 1:N company_contacts`。不使用 DB ENUM。

## 3. `roles` 🟡

角色主檔。已同意角色可由公司／系統建立，不把老闆、HR、主管固定寫死；完整欄位未最終確認。至少需要 id、company_id 或系統範圍、code、name、is_active、timestamps。

## 4. `permissions` 🟡

功能權限主檔。已同意支援 parent → child → child 階層；完整欄位未最終確認。至少包含 id、parent_id、code、name、排序及啟用狀態。

## 5. `role_permissions` 🟡

角色與權限 N:N 關聯。核心鍵為 role_id、permission_id；是否包含授權範圍與操作人尚未定案。

## 6. `departments` 🟡

公司部門及樹狀組織。已同意可表達總經理室 → 研發部 → 研發一處；完整欄位未定案。至少需要 id、company_id、parent_id、code、name、is_active。

## 7. `employees` ✅

**用途：** 保存「人」，不保存任職狀態。

| 欄位 | 說明 |
|---|---|
| id | PK |
| company_id | 所屬公司 |
| employee_code | 員工編號 |
| name | 姓名 |
| gender | 性別代碼 |
| identity | 身分識別資料，加密 |
| birthday | 加密 |
| phone | 加密 |
| email | 加密 |
| address | 加密 |
| created_at/updated_at | 時間欄位 |

**明確不放：** `status`、`employment_sequence`。離職回任不新增另一個 employees。

## 8. `employee_employments` ✅

| 欄位 | 說明 |
|---|---|
| id | PK |
| employee_id | FK → employees |
| employment_type_code | 聘僱類型 |
| employment_nature_code | 聘僱性質 |
| hire_date | 到職日 |
| leave_date | 離職日 |
| leave_reason_code | 離職原因 |
| status | 本次任職狀態 |
| created_at/updated_at | 時間欄位 |

一個員工可有多段任職歷史。

## 9. `employee_department_histories` ✅

employee_id、department_id、effective_from、effective_to。員工同一時間只能有一個有效部門；調部門新增歷史，不覆蓋舊紀錄。

## 10. `job_titles` 🟡

職稱主檔，例如工程師、經理、協理。表概念已定；完整欄位未最後確認。

## 11. `employee_job_title_histories` ✅

employee_id、job_title_id、effective_from、effective_to。升遷不得只 UPDATE `employees.job_title_id`。

## 12. `job_positions` 🟡

職務主檔。職務與職稱分離；完整欄位待定。

## 13. `employee_job_position_histories` ✅

employee_id、job_position_id、effective_from、effective_to。同一員工同時間可有多個職務。

## 14. `employee_dependents` ✅／欄位部分待補

保存配偶、扶養親屬及列報扶養生效歷史。親屬關係採 int code，說明放程式註釋／文件，不另建 `dependent_relationships`。對話已提及敏感姓名、身分證、生日及 is_dependent、生效期間，但完整型態與必要性未逐欄最終確認。

## 15. `employee_withholding_settings` ✅／欄位部分待補

保存員工每月經常性薪資扣繳方式與生效歷史。核心為 employee_id、withholding_method_code、effective_from、effective_to。

- 1：依薪資所得扣繳稅額表
- 2：按全月給付總額 5%

經常性／非經常性不是員工屬性，由薪資項目決定。

## 16. `employee_salary_bank_accounts` 🟡

發薪銀行帳戶。已討論銀行、分行、加密帳號、帳號 Hash、加密戶名、生效／主要帳戶；正式欄位與是否容許多帳戶尚未最後確認。
