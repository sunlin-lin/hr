# HR Schema 定案恢復總索引

> 依原對話中使用者明確表示「記錄起來／定案／確認／可以」且語境指向 Schema 的節點恢復。恢復日期：2026-08-23。

## 採納標準

- **A｜逐欄已確認**：可恢復表名、欄位與型態。
- **B｜結構／規則已確認**：表或模組存在、核心規則確定，但部分欄位或型態未可靠恢復。
- **C｜已討論、未定案**：不得作為開發依據。
- 不使用 DB ENUM；歷史資料不以目前狀態覆蓋；敏感明文加密後用 BINARY/VARBINARY，需要查詢者另存 binary hash。
- 未能可靠恢復者一律標示「已確認存在／欄位待恢復」，不補猜測。

## 文件

1. `01-company-organization-employee.md`：公司、角色權限、組織人事。
2. `02-scheduling-attendance.md`：排班與出勤；含大量暫稱，須依本索引的衝突說明使用。
3. `03-business-modules.md`：加班、補休、請假、公司贈與假、薪資、人事成本。
4. `04-regulatory-system-audit.md`：法規同步、系統管理與舊稿衝突。

## 恢復狀態

| 主題 | 狀態 | 說明 |
|---|---|---|
| 公司 | A | Company Schema v1 有明確確認節點 |
| 角色／權限 | A/B | 三表與結構可恢復；不寫死角色功能 |
| 組織／人事 | A/B | 歷史模型已確認；部分欄位型態待恢復 |
| 排班／出勤 | B | 核心規則確認；多個表名只是暫稱 |
| 加班／補休 | B | 表名與規則確認；完整逐欄待恢復 |
| 請假／公司贈與假 | B | 表名與規則確認；完整逐欄待恢復 |
| 薪資 | B | 原則與部分表名確認；主體欄位待恢復 |
| 人事成本 | B | 模組確認；正式表名與欄位待恢復 |
| 法規同步 | A | 四核心表及逐欄 Schema 已定案 |
| 系統管理 | B | 角色權限確定；稽核日誌欄位待恢復 |

## 不得誤用

- `employee_departments` 已更正為 `employee_department_histories`。
- `employee_positions` 已更正為職稱／職務分離的歷史模型。
- `employee_holiday_calendars` 明確否決。
- 排班稿的 `schedule_patterns`、`schedule_pattern_days`、`employee_schedule_assignments`、`employee_schedule_changes`、`attendance_cancellations`、`attendance_correction_requests`、`attendance_correction_approvals` 都是暫稱，不是定案表名。


