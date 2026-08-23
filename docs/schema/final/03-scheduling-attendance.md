# 班別、排班、出勤

## `shifts` ✅

班別主檔。

| 欄位 | 說明 |
|---|---|
| id | PK |
| company_id | 公司 |
| code/name | 班別代碼／名稱 |
| start_time/end_time | 預定上下班時間 |
| break_minutes | 休息總分鐘 |
| cross_day | 是否跨日 |
| is_active | 啟用狀態 |
| created_at/updated_at | 時間欄位 |

班別是時段定義，不是每日實際班表；必須支援 22:00 至翌日 06:00 等跨日班。

## `schedule_patterns` 🟡（暫定名稱）

固定週班、循環班、輪班的規則主體。欄位方向：id、company_id、code、name、pattern_type_code、cycle_length、is_active、timestamps。

定案規則：做二休二不得硬塞進星期一至星期日；週期不必等於 7 天。

## `schedule_pattern_days` 🟡（暫定名稱）

週期第 N 天的班別與工作日性質。欄位方向：id、schedule_pattern_id、sequence_no、shift_id、day_type_code、timestamps。

## `employee_schedule_assignments` 🟡（暫定名稱）

員工在某期間套用哪一套排班規則。欄位方向：id、employee_id、schedule_pattern_id、effective_from、effective_to、cycle_anchor_date、timestamps。

## `employee_schedules` ✅

每日實際班表。

| 欄位 | 說明 |
|---|---|
| id | PK |
| employee_id | 員工 |
| schedule_date | 排班日期 |
| shift_id | 班別；休息日可空 |
| start_at/end_at | 該日實際預定區間 |
| day_type_code | 工作日／休息日／例假／假日 |
| source_type_code | 規則產生／人工／調班 |
| created_at/updated_at | 時間欄位 |

定案：預先產生並持續延伸；歷史不可被模板覆蓋；請假不刪班表；加班資格依本表判定；零工可直接建立本表。

## `employee_schedule_changes` 🟡（暫定名稱）

保存調班前後資料、原因、操作者與時間，不允許無痕 UPDATE。欄位方向：employee_schedule_id、before/after shift 與時段、reason、changed_by、changed_at。

## `attendance_records` ✅

| 欄位 | 說明 |
|---|---|
| id | PK |
| employee_id | 員工 |
| employee_schedule_id | 可關聯當日班表 |
| punch_type_code | 上班／下班 |
| punched_at | 打卡時間 |
| latitude/longitude | GPS，可空 |
| source_type_code | 實際打卡／補登 |
| created_at | 建立時間 |

定案：下班卡必須先有上班卡；GPS 非強制；打卡不可反向修改班表。

## `attendance_cancellations` 🟡（暫定名稱）

attendance_record_id、cancelled_by、cancelled_at、reason、created_at。上下班卡均可撤銷；撤銷不 DELETE 原卡。

## `attendance_correction_requests` 🟡（暫定名稱）

employee_id、employee_schedule_id、punch_type_code、requested_punched_at、reason、status_code、submitted_at、timestamps。上班與下班補登分開申請。

## `attendance_correction_approvals` 🟡（暫定名稱）

attendance_correction_request_id、action_code、acted_by、acted_at、reason、created_at。核准後形成可追溯原申請的有效補登結果。

## `attendance_results` 🟡

每日出勤計算結果概念已討論，可能包含 scheduled/worked/late/early_leave/absence/overtime minutes、result_status_code、calculated_at；表名與是否保存為實體結果尚未最終同意。

## 出勤異常 Schema 🟡

責任已同意：缺上班卡、缺下班卡、遲到、早退、班表不一致，以及由補登、請假或調班解除異常。正式名稱與欄位未定案。
