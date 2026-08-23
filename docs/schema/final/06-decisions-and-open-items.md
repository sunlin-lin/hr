# 否決方案、名稱修正與待確認事項

## 明確否決／取代

| 舊方案 | 最終處理 |
|---|---|
| `employee_departments` | 改為 `employee_department_histories` |
| `employee_positions` | 職稱、職務分離，改由 job title／job position histories |
| `employees.status` | 不採用；狀態放任職關係 |
| `employees.employment_sequence` | 不採用 |
| `employees.job_title_id` 直接 UPDATE | 不採用；使用歷史表 |
| `company_addresses` | 不採用；地址直接放 companies |
| `dependent_relationships` | 不採用；關係使用 int code 與文件註釋 |
| `employee_holiday_calendars` | 不採用；工作／假日依 employee_schedules |
| 打卡 DELETE | 不採用；使用撤銷紀錄 |
| 一筆加班拆成加班費＋補休 | 不採用；只能二選一 |
| `regulatory_sources` | 不採用；來源與 Adapter 由程式碼管理 |
| `regulatory_datasets` | 不採用；支援資料類型由程式碼固定 |
| DB ENUM | 全系統不採用 |

## 早期過度拆分、未列為正式方案

- shift_definitions、shift_work_periods、shift_breaks、shift_rules
- schedule_cycles、schedule_cycle_details、schedule_periods
- schedule_rules、schedule_rule_details、schedule_exceptions
- schedule_change_requests、schedule_change_details
- regulatory_rules、regulatory_rule_versions、regulatory_rule_parameters
- occupational_industry_categories、insurance_unit_types

這些不代表永遠不能建立；目前沒有取得「全部保留」的最終同意，正式方案先採收斂架構。

## 尚待最後確認

1. roles、permissions、departments 的完整欄位及公司／系統共用範圍。
2. job_titles、job_positions 主檔完整欄位。
3. employee_dependents 完整敏感欄位與文件表需求。
4. employee_salary_bank_accounts 是否允許多筆同時有效。
5. 薪資設定、薪資項目、人事成本完整逐欄 Schema。
6. schedule_patterns 等四張排班規則表的正式名稱。
7. 是否保留獨立 shift_breaks，而非 shifts.break_minutes。
8. attendance_results 是否為實體 Snapshot 表。
9. 出勤異常正式表名與解除流程。
10. leave request 是否允許同一主單混合多個假別。
11. payroll_regulatory_snapshots 獨立表或直接關聯 settlement。
12. audit_logs 正式名稱、保存期限與敏感資料遮蔽。
