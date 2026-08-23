# 整體布局與資料流

## 系統定位

本系統是多公司 HR SaaS，範圍包含公司與權限、組織人事、排班出勤、加班補休、請假假額度、薪資結算、政府法規同步及稽核。

## 全域定案原則

1. 主鍵主要採 UUID。
2. 不使用 DB ENUM；類型與狀態使用一般 code/string/int。
3. 敏感資料加密；需要查詢比對者另存 Hash。
4. 歷史資料不以目前狀態覆蓋。
5. 具生效時間的資料使用 `effective_from/effective_to`。
6. 核發、審核、撤銷、結算都保留不可抹除歷史。
7. 結算資料建立 Snapshot，後續規則或薪資變更不回寫歷史。

## 主資料布局

```text
companies
├─ company_contacts
├─ roles ── role_permissions ── permissions
├─ departments
└─ employees
   ├─ employee_employments
   ├─ employee_department_histories
   ├─ employee_job_title_histories
   ├─ employee_job_position_histories
   ├─ employee_dependents
   ├─ employee_withholding_settings
   ├─ employee_salary_settings / histories / items
   └─ employee_salary_bank_accounts
```

## 排班到薪資

```text
shifts
 ↓
schedule_patterns / schedule_pattern_days
 ↓
employee_schedule_assignments
 ↓
employee_schedules
 ├─ attendance_records
 ├─ leave_requests
 ├─ overtime_requests
 └─ payroll_settlements
```

- 班表是「應工作事實」。
- 打卡是「實際到離事實」。
- 請假是「原應工作但經核准免除」。
- 加班是「原班表以外且經申請核准的工作」。
- 四者不可合併成同一張表。

## 加班與補休

```text
overtime_requests
 ↓
overtime_approvals
 ↓
overtime_compensations
 ├─ 加班費 → payroll_pending_items
 └─ 補休
      ├─ compensatory_leave_credits
      ├─ compensatory_leave_rate_snapshots
      ├─ compensatory_leave_transactions
      └─ compensatory_leave_allocations
             ↓ 到期未使用
        payroll_pending_items
```

## 請假與額度

```text
leave_types
 ↓
leave_type_rules
 ↓
leave_entitlements
 ├─ leave_balances
 └─ leave_balance_transactions
          ↑
leave_requests
 ├─ leave_request_details
 ├─ leave_request_approvals
 ├─ leave_request_allocations
 └─ leave_request_documents
```

## 法規到 Payroll

```text
政府資料
 ↓
regulatory_dataset_versions
 ├─ raw_format_code + raw_data
 └─ regulatory_records
          ↓
Payroll 依適用日期取版本
          ↓
payroll_settlements
 └─ 法規版本 Snapshot
```
