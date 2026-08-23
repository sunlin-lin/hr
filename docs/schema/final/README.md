# HR 系統 Schema 最終決策整理

> 來源：2026-08-22 至 2026-08-23「整理人資系統需求」共 149 個對話回合。  
> 原始對話：[`docs/conversations/2026-08-22-to-2026-08-23/`](../../conversations/2026-08-22-to-2026-08-23/)  
> 整理規則：後面的明確修正覆蓋前面的方案；沒有取得使用者最終同意的名稱或欄位，不補成假定案。

## 狀態標記

- ✅ **已定案**：對話中有明確同意或後續反覆沿用。
- 🟡 **概念已定、Schema 未完全定案**：責任與關係已同意，但正式表名或完整欄位仍缺最後確認。
- ❌ **否決／被取代**：不應實作成目前正式方案。

## 文件

1. [整體布局與資料流](./00-architecture.md)
2. [公司、權限、組織、員工](./01-company-organization-employee.md)
3. [薪資設定與人事成本](./02-salary-and-cost.md)
4. [班別、排班、出勤](./03-scheduling-attendance.md)
5. [加班、補休、請假、公司贈與](./04-overtime-leave.md)
6. [Payroll、政府法規、系統管理](./05-payroll-regulatory-system.md)
7. [否決方案、名稱修正與待確認事項](./06-decisions-and-open-items.md)

## 最終核心 Schema 骨架

```text
公司／權限
companies
company_contacts
roles
permissions
role_permissions

組織／員工
departments
employees
employee_employments
employee_department_histories
job_titles
employee_job_title_histories
job_positions
employee_job_position_histories
employee_dependents
employee_withholding_settings
employee_salary_bank_accounts

薪資制度
payroll_settings
salary_items
employee_salary_settings
employee_salary_histories
employee_salary_items
payroll_periods
payroll_details
payroll_payments
personnel_costs
personnel_cost_items

排班／出勤
shifts
schedule_patterns                    🟡
schedule_pattern_days                🟡
employee_schedule_assignments        🟡
employee_schedules
employee_schedule_changes            🟡
attendance_records
attendance_cancellations             🟡
attendance_correction_requests       🟡
attendance_correction_approvals      🟡
attendance_results                   🟡

加班／補休
overtime_requests
overtime_approvals
overtime_compensations
compensatory_leave_credits
compensatory_leave_rate_snapshots
compensatory_leave_transactions
compensatory_leave_allocations

請假／假額度
leave_types
leave_type_rules
leave_entitlements
leave_balances
leave_balance_transactions
leave_requests
leave_request_details
leave_request_approvals
leave_request_allocations
leave_request_documents
leave_events
company_leave_grant_batches
company_leave_grants

Payroll
payroll_pending_items
payroll_settlements
payroll_regulatory_snapshots          🟡

政府法規
company_regulatory_settings
regulatory_dataset_versions
regulatory_records
regulatory_sync_logs

系統管理
audit_logs                            🟡（暫稱）
```
