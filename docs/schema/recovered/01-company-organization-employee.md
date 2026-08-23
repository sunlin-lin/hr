# 已恢復 Schema：公司／組織／員工

> 狀態：從既有討論中可可靠恢復的第一批 Schema。只記錄已確認內容；不以推測補齊未恢復欄位。

## 全域設計原則

- 多公司 HR SaaS。
- 主鍵主要採 UUID。
- 不使用 DB ENUM；業務狀態與類型使用一般 code/string/int 欄位。
- 可變業務規則不綁死資料庫 ENUM。
- 敏感資料需要加密；需要搜尋的敏感值另保存 Hash。
- 歷史資料原則上保留，不以目前狀態覆蓋歷史。
- 每張正式 Schema 後續統一文件格式：用途說明、每個欄位說明、型態、必要性、關聯、設計理由。

---

## 1. `companies`

### 用途

SaaS Tenant／公司主體。

### 已恢復欄位

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | UUID | 必填 | 主鍵 |
| `company_code` | STRING | 必填 | 系統公司唯一編號，全域唯一 |
| `company_type` | STRING | 必填 | 公司／個人等主體類型 |
| `legal_type` | STRING | 必填 | 法律型態 |
| `tax_id` | STRING | 視類型 | 統一編號等識別 |
| `name` | STRING | 必填 | 正式名稱 |
| `short_name` | STRING | 選填 | 簡稱 |
| `status` | STRING | 必填 | 公司狀態 |
| `registered_postal_code` | STRING | 選填 | 登記地址郵遞區號 |
| `registered_city` | STRING | 選填 | 登記縣市 |
| `registered_district` | STRING | 選填 | 登記行政區 |
| `registered_address` | STRING | 選填 | 登記地址 |
| `actual_postal_code` | STRING | 選填 | 實際地址郵遞區號 |
| `actual_city` | STRING | 選填 | 實際縣市 |
| `actual_district` | STRING | 選填 | 實際行政區 |
| `actual_address` | STRING | 選填 | 實際地址 |
| `invoice_postal_code` | STRING | 選填 | 發票地址郵遞區號 |
| `invoice_city` | STRING | 選填 | 發票縣市 |
| `invoice_district` | STRING | 選填 | 發票行政區 |
| `invoice_address` | STRING | 選填 | 發票地址 |
| `created_at` | DATETIME | 必填 | 建立時間 |
| `updated_at` | DATETIME | 必填 | 更新時間 |
| `deleted_at` | DATETIME | 選填 | Soft Delete |

### 已確認規則

- 地址直接放 `companies`，不另外拆 address 表。
- `company_code` 與 `id` 分開。
- `company_code` 全域唯一。
- 公司型主體：`統編 + 3 碼流水號`，例如 `12345678001`。
- 個人型主體：`YYYYMMDD + 3 碼流水號`，例如 `19890101001`。
- `company_code` 不使用分隔符號。
- 流水號只增不減。

---

## 2. `company_contacts`

### 用途

保存公司負責人、業務、帳務等聯絡資訊。

### 已恢復欄位

| 欄位 | 必要性／說明 |
|---|---|
| `id` | 主鍵 |
| `company_id` | 所屬公司 |
| `contact_type` | 聯絡人類型 |
| `name` | 姓名 |
| `identity` | 身分識別資料，加密 |
| `identity_hash` | 身分識別查詢 Hash |
| `birthday` | 生日，加密 |
| `phone` | 電話，加密 |
| `phone_hash` | 電話查詢 Hash |
| `email` | Email，加密 |
| `email_hash` | Email 查詢 Hash |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

### 已確認關聯

`companies 1:N company_contacts`

### 已討論聯絡類型

- `OWNER`
- `SALES`
- `ACCOUNTING`

不使用 DB ENUM。

---

## 3. `roles`

### 用途

角色主檔。角色本身不把老闆、人資、主管等固定寫死在 Schema。

目前完整欄位尚待從原討論繼續恢復。

---

## 4. `permissions`

### 用途

功能權限主檔。

### 已確認規則

權限支援階層：`parent -> child -> child`。

完整欄位尚待繼續恢復。

---

## 5. `role_permissions`

### 用途

`roles` 與 `permissions` 的 Many-to-Many 關聯。

`roles N:N permissions`

完整欄位尚待繼續恢復。

---

## 6. `departments`

### 用途

公司部門／組織主檔。

已確認系統有部門及組織階層概念；完整欄位待繼續恢復。

---

## 7. `employees`

### 用途

員工「人」的主檔。任職狀態不直接塞在員工主表。

### 已恢復欄位

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `company_id` | 所屬公司 |
| `employee_code` | 員工編號 |
| `name` | 姓名 |
| `gender` | 性別 |
| `identity` | 身分識別資料；敏感資料 |
| `birthday` | 生日；敏感資料 |
| `phone` | 電話；敏感資料 |
| `email` | Email；敏感資料 |
| `address` | 地址；敏感資料 |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

### 明確不放

- `status`
- `employment_sequence`

### 設計理由

「人」與「任職關係」分離。員工離職後再次回任，不需要重新建立另一個 `employees`。

---

## 8. `employee_employments`

### 用途

保存員工到職、離職、重新任職等任職歷史。

### 已恢復欄位

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `employee_id` | 員工 |
| `employment_type_code` | 聘僱類型 |
| `employment_nature_code` | 聘僱性質 |
| `hire_date` | 到職日期 |
| `leave_date` | 離職日期 |
| `leave_reason_code` | 離職原因代碼 |
| `status` | 任職狀態 |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

---

## 9. `employee_department_histories`

### 用途

保存員工部門異動歷史。

### 已恢復核心欄位

| 欄位 | 說明 |
|---|---|
| `employee_id` | 員工 |
| `department_id` | 部門 |
| `effective_from` | 生效開始 |
| `effective_to` | 生效結束；目前有效可為 NULL |

### 核心規則

- 同一員工在同一時間只能有一個有效部門。
- 調部門不可覆蓋舊資料，必須保留歷史。
- 正式名稱為 `employee_department_histories`，不是 `employee_departments`。

---

## 10. `job_titles`

### 用途

職稱主檔，例如工程師、資深工程師、經理、協理。

完整欄位待繼續恢復。

---

## 11. `employee_job_title_histories`

### 用途

保存員工職稱／升遷歷史。

### 已恢復核心欄位

| 欄位 | 說明 |
|---|---|
| `employee_id` | 員工 |
| `job_title_id` | 職稱 |
| `effective_from` | 生效開始 |
| `effective_to` | 生效結束；目前有效可為 NULL |

### 核心規則

不可只在 `employees.job_title_id` 上 UPDATE，否則升遷歷史會消失。

---

## 12. `job_positions`

### 用途

職務主檔。職務與職稱為不同概念。

完整欄位待繼續恢復。

---

## 13. `employee_job_position_histories`

### 用途

保存員工職務及其歷史。

### 已恢復核心欄位

| 欄位 | 說明 |
|---|---|
| `employee_id` | 員工 |
| `job_position_id` | 職務 |
| `effective_from` | 生效開始 |
| `effective_to` | 生效結束 |

### 核心規則

- 部門：同一時間只能有一個。
- 職務：同一時間可以有多個。

例如同一員工可同時為 `Backend Developer` 與 `Project Leader`。

---

## 14. `employee_dependents`

### 用途

保存員工配偶／扶養親屬與列報扶養的生效歷史。

### 已確認規則

- 親屬關係採 `int code`。
- 代碼說明放程式註釋／文件，不另外建立親屬關係代碼表。
- 列報扶養需要生效歷史，供薪資所得扣繳計算使用。

完整欄位待從原討論繼續恢復。

---

## 15. `employee_withholding_settings`

### 用途

保存員工每月經常性薪資的所得稅扣繳方式及生效歷史。

### 已確認代碼

- `1` = 依薪資所得扣繳稅額表
- `2` = 按全月給付總額 5%

### 核心規則

- 扣繳方式需要生效歷史。
- 經常性／非經常性薪資不是員工屬性，由薪資項目的給付性質決定。

完整欄位待從原討論繼續恢復。

---

## 本批恢復狀態

本文件只代表目前已可靠恢復的「公司／組織／員工」部分。尚未完整恢復的欄位明確標記為待恢復，不自行補欄位。

下一批預計恢復：班別／排班／出勤。