# 班別、排班與出勤

## 設計理念

規則 → 排定 → 異動 → 實際 → 判定。班別不是班表；班表不是打卡；請假不修改班表。週期不限制七天，支援做二休二、輪班、零工、跨日、臨時叫班、換班、公司停班與調班歷史。

## 排班 Schema

### `shift_definitions`

**註釋：** 班別主檔。

`id uuid PK`、`company_id uuid FK`、`code varchar`、`name varchar`、`work_type_code integer`、`is_overnight boolean`、`is_flexible boolean`、`required_work_minutes integer`、`description text`、`is_active boolean`、`created_at datetime`、`updated_at datetime`、`deleted_at datetime nullable`。

### `shift_work_periods`

**註釋：** 班別內實際工作區段，支援一天多段及跨日。核心欄位：`id`、`shift_definition_id`、`sequence_no`、`start_time`、`end_time`、`end_day_offset`、`work_minutes`、時間欄位。

### `shift_breaks`

**註釋：** 班別休息區段。核心欄位：`id`、`shift_definition_id`、`sequence_no`、`start_time`、`end_time`、`break_minutes`、`is_paid`、時間欄位。

### `schedule_rules` / `schedule_rule_details`

**註釋：** 固定週班或任意長度循環規則及每個週期日內容。規則需含公司、名稱、類型、週期長度與啟用狀態；明細需含週期序號、班別、日期性質及是否安排工作。做二休二以四日週期表達，不綁星期。

### `employee_schedule_assignments`

**註釋：** 員工在有效期間套用哪一套排班規則，包含週期定位基準日。正式名稱在原對話中未單獨再確認，概念與欄位責任已確認。

### `schedule_periods`

`id uuid PK`、`company_id uuid FK`、`name string`、`start_date date`、`end_date date`、`status_code integer`、`published_at datetime nullable`、`published_by uuid nullable`、`created_at datetime`、`updated_at datetime`。

### `employee_schedules`

**註釋：** 員工某日最終有效班表快照；排班確定／發布時產生。

核心欄位：`id`、`schedule_period_id`、`company_id`、`employee_id`、`employment_id`、`schedule_date`、`shift_definition_id nullable`、`schedule_day_type_code integer`、`scheduled_work_flag boolean`、預定起訖時間、來源代碼、狀態及時間欄位。

規則：歷史班表不得被新規則覆蓋；零工可直接建立；國定假日不等於每個員工休假；加班資格由該日班表性質與是否排定工作共同判定；不建立 `employee_holiday_calendars`。

### `schedule_changes`

**註釋：** 已發布班表異動／調班歷史。

`id uuid PK`、`employee_schedule_id uuid FK`、`employee_id uuid FK`、`original_shift_id uuid nullable`、`new_shift_id uuid nullable`、原／新日期性質、原／新是否工作、`reason string`、`status_code integer`、`requested_by uuid`、`approved_by uuid nullable`、`approved_at datetime nullable`、`effective_at datetime`、`created_at datetime`、`updated_at datetime`。

## 出勤 Schema

### `attendance_records`

**註釋：** 正常或核准補登形成的正式打卡事件。

`id uuid PK`、`employee_id uuid FK`、`employment_id uuid FK`、`employee_schedule_id uuid nullable FK`、`work_date date`、`attendance_type_code integer`（1上班、2下班）、`clocked_at datetime`、`latitude decimal nullable`、`longitude decimal nullable`、`source_type_code integer`、撤銷狀態／時間／人員及建立時間欄位。

規則：有效上班卡後才能打下班卡；兩種卡均可撤銷；撤銷不 DELETE；GPS 選填。

### `attendance_correction_requests`

**註釋：** 忘打卡補登申請；上班與下班分開申請。

核心欄位：`id`、`employee_id`、`employment_id`、`employee_schedule_id`、`attendance_type_code`、`requested_clocked_at`、`reason`、`status_code`、送出／核准／拒絕人員與時間、建立／修改時間。

### `attendance_settings`

**註釋：** 公司打卡規則；GPS 是否啟用不得等同強制必填。

### `attendance_results`

**註釋：** 依班表、有效打卡、請假與異動計算的遲到、早退、缺卡等判定結果；不得反向改寫原始打卡或班表。


