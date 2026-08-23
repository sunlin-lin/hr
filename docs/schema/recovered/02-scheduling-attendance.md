# 已恢復 Schema：班表／排班／出勤（補完整版本）

> 狀態：第二批恢復。上一版只恢復到 `employee_schedules` 與 `attendance_records`，確實漏掉排班規則層、週期層、員工套用層、班表異動層、補登與審核層。本版把目前可由既有討論可靠恢復的 Schema 結構補回來；無法確認「昨天最終正式命名」者會明確標示名稱仍待核對，不把猜測當定案。

## 本批應包含的完整模組

```text
班別
├─ 班別主檔
└─ 班別時間／跨日設定

排班體系
├─ 固定週班規則
├─ 循環班規則（做二休二等）
├─ 循環節點／週期日
├─ 員工排班規則套用期間
├─ 員工實際班表
└─ 班表異動／調班歷史

出勤
├─ 打卡紀錄
├─ 打卡撤銷
├─ 補登申請
├─ 補登審核
└─ 出勤異常
```

---

# 一、`shifts`（班別主檔）

## 用途

定義可被班表使用的「班別」，例如日班、晚班、夜班。

## 可可靠恢復的欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `company_id` | 所屬公司 |
| `code` | 班別代碼 |
| `name` | 班別名稱 |
| `start_time` | 預定上班時間 |
| `end_time` | 預定下班時間 |
| `break_minutes` | 休息分鐘數／休息總時數 |
| `cross_day` | 是否跨日班 |
| `is_active` | 是否啟用 |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

## 已確認規則

- 班別只是「工作時段定義」，不是員工每天的實際班表。
- 必須支援跨日班，例如 22:00 ～ 翌日 06:00。
- 班表會引用班別後，再產生員工實際每日排班。

---

# 二、排班規則主體（正式表名待核對）

原討論明確存在「班表體系／排班規則」概念，且不能只用星期一～星期日。

目前恢復出的責任是：

```text
排班規則
├─ 固定週班
├─ 循環班
├─ 輪班
└─ 零工／非固定排班
```

建議暫以 `schedule_patterns` 表示此概念；**正式名稱需再從原討論核對**。

## 可恢復核心欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `company_id` | 所屬公司 |
| `code` | 排班規則代碼 |
| `name` | 排班規則名稱 |
| `pattern_type_code` | 固定週班／循環班／輪班等類型 |
| `cycle_length` | 循環班週期長度；固定週班可為 7 |
| `is_active` | 是否啟用 |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

## 已確認規則

- 「做二休二」不可硬塞進一週 7 天模型。
- 週期可以大於、等於或不依賴 7 天。
- 零工／非固定班可以不依賴長期模板，而直接形成實際班表。

---

# 三、排班週期節點／週期日（正式表名待核對）

這一層是上一版漏掉的重要 Schema。

## 用途

描述一個排班週期裡第 N 天是上哪個班、還是休息日。

例如做二休二：

```text
第 1 天 → 工作班
第 2 天 → 工作班
第 3 天 → 休息
第 4 天 → 休息
然後循環
```

建議暫以 `schedule_pattern_days` 表示；**正式名稱待核對**。

## 可恢復欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `schedule_pattern_id` | 所屬排班規則 |
| `sequence_no` | 週期第幾天 |
| `shift_id` | 當日班別；休息日可空 |
| `day_type_code` | 工作日／休息日／例假／其他排班性質 |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

## 設計理由

這張就是用來解決「一週只有 7 天，做二休二怎麼設計」的問題。

---

# 四、員工排班規則套用期間（正式表名待核對）

這也是上一版漏掉的一層。

## 用途

記錄某員工在某段期間使用哪一套排班體系，並提供實際班表產生器依據。

建議暫以 `employee_schedule_assignments` 表示；**正式名稱待核對**。

## 可恢復欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `employee_id` | 員工 |
| `schedule_pattern_id` | 套用的排班規則 |
| `effective_from` | 套用開始日 |
| `effective_to` | 套用結束日 |
| `cycle_anchor_date` | 循環班從週期第 1 天開始計算的基準日 |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

## 已確認規則

- 同一員工可以隨時間換不同班表體系。
- 變更未來規則不能回頭改寫已產生的歷史 `employee_schedules`。
- 循環班必須有可以定位週期位置的基準點。

---

# 五、`employee_schedules`

## 用途

保存員工「某一天實際被排到什麼班」，是後續出勤、請假、加班資格判斷的重要事實資料。

## 已恢復核心欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `employee_id` | 員工 |
| `schedule_date` | 實際排班日期 |
| `shift_id` | 當日班別；休息日可空 |
| `start_at / start_time` | 當日預定上班時間 |
| `end_at / end_time` | 當日預定下班時間 |
| `day_type_code` | 工作日／休息日／例假／假日等班表性質 |
| `source_type_code` | 由規則產生／人工排班／調班等來源方向 |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

## 核心規則

- `employee_schedules` 是「實際班表」，不是永久排班規則。
- 系統會預先產生一段期間的員工班表，之後再持續延伸。
- 歷史已產生班表不能因模板修改而被覆蓋。
- 員工請假不刪除原班表。
- 加班資格要依該日 `employee_schedules` 的班表性質判定。
- 原討論否決 `employee_holiday_calendars` 這類另建員工假日日曆的設計；假日／工作日性質直接由實際班表體系判定。

---

# 六、班表異動／調班歷史（正式表名待核對）

## 用途

班已經產生後，中途調班時，保留原排班與異動歷史。

建議暫以 `employee_schedule_changes` 表示；**正式名稱待核對**。

## 可恢復核心欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `employee_schedule_id` | 被異動的實際班表 |
| `before_shift_id` | 原班別 |
| `after_shift_id` | 新班別 |
| `before_start_at` | 原預定上班時間 |
| `before_end_at` | 原預定下班時間 |
| `after_start_at` | 新預定上班時間 |
| `after_end_at` | 新預定下班時間 |
| `reason` | 調班原因 |
| `changed_by` | 異動者 |
| `changed_at` | 異動時間 |

## 已確認規則

- 調班不能無痕 UPDATE。
- 實際班表可以形成新狀態，但原班表內容／異動原因必須可追蹤。

---

# 七、零工／非固定排班

## 已確認概念

零工、臨時工、非固定工時員工不一定有固定週班或循環模板。

這種情況可以直接建立 `employee_schedules`：

```text
指定日期
+ 指定班別／時段
+ day_type_code
```

因此**不需要為零工額外建立完全不同的一套班表表格**；是否由規則產生或人工直接排入，可由來源欄位／排班流程區分。

---

# 八、`attendance_records`

## 用途

保存員工實際上下班打卡事件。

## 已恢復核心欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 打卡紀錄 ID |
| `employee_id` | 員工 |
| `employee_schedule_id` | 可關聯當日實際班表 |
| `punch_type_code` | `上班`／`下班` |
| `punched_at` | 實際打卡時間 |
| `latitude` | GPS 緯度，可空 |
| `longitude` | GPS 經度，可空 |
| `source_type_code` | 員工實際打卡／補登等來源方向 |
| `created_at` | 建立時間 |

## 已確認規則

- 上班打卡後才能打下班卡。
- GPS 非強制。
- 打卡和班表是不同事實資料。
- 打卡不能反向改寫排班。

---

# 九、打卡撤銷紀錄（正式表名待核對）

上一版只寫了規則，沒有把 Schema 層補回來。

## 用途

保存員工自行撤銷誤打卡的行為與歷史，不 DELETE 原打卡。

建議暫以 `attendance_cancellations` 表示；**正式名稱待核對**。

## 可恢復欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `attendance_record_id` | 被撤銷的打卡 |
| `cancelled_by` | 撤銷者；可為員工本人 |
| `cancelled_at` | 撤銷時間 |
| `reason` | 撤銷原因，可選填 |
| `created_at` | 建立時間 |

## 核心規則

- 上班卡、下班卡都可以撤銷。
- 撤銷不是 DELETE。
- 原始打卡永遠保留。

---

# 十、補登申請（正式表名待核對）

## 用途

員工忘記打卡時申請補登。

建議暫以 `attendance_correction_requests` 表示；**正式名稱待核對**。

## 可恢復核心欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `employee_id` | 申請員工 |
| `employee_schedule_id` | 對應班表 |
| `punch_type_code` | 補上班卡／補下班卡 |
| `requested_punched_at` | 申請補登的時間 |
| `reason` | 補登原因 |
| `status_code` | 待審／核准／拒絕等流程狀態 |
| `submitted_at` | 送出時間 |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

## 核心規則

- 上班補登與下班補登分開申請。
- 員工可能只漏一張卡，不要求一次補完整天。
- 補登不能直接修改原始打卡資料。

---

# 十一、補登審核（正式表名待核對）

## 用途

保存補登申請的審核歷史。

建議暫以 `attendance_correction_approvals` 表示；**正式名稱待核對**。

## 可恢復欄位方向

| 欄位 | 說明 |
|---|---|
| `id` | 主鍵 |
| `attendance_correction_request_id` | 補登申請 |
| `action_code` | 核准／拒絕 |
| `acted_by` | 審核者 |
| `acted_at` | 審核時間 |
| `reason` | 拒絕／審核說明 |
| `created_at` | 建立時間 |

## 核心規則

核准後產生／形成有效的補登出勤結果；補登來源仍要能追溯到原申請。

---

# 十二、出勤異常（Schema 名稱待核對）

原討論已明確提過「出勤異常」這個模組，但目前完整正式表名與欄位仍未恢復。

已確認異常來源至少包括：

- 已過應上班時間但沒有有效上班卡
- 有上班卡但缺下班卡
- 打卡與班表時段不一致所形成的異常判斷
- 後續可能由補登／請假／調班等流程解除異常

此處先恢復模組責任，不自行宣告完整 Schema。

---

# 十三、班表／出勤與其他模組的相依關係

```text
shifts
   ↓
排班規則／週期
   ↓
員工規則套用
   ↓
employee_schedules
   │
   ├──→ attendance_records
   │       ├──→ 打卡撤銷
   │       └──→ 補登申請／審核
   │
   ├──→ leave_requests
   │       └── 計算原本應工作時數
   │
   └──→ overtime_requests
           └── 判定工作日／休息日／假日加班資格
```

重要原則：

- 班表 = 應工作事實。
- 打卡 = 實際到離事實。
- 請假 = 原本應工作、但經核准免除工作。
- 加班 = 原班表以外、符合申請與核准條件的工作。
- 四者不得合併為同一份資料。

---

# 本批目前恢復出的 Schema 清單

### 名稱可確認／高可信

- `shifts`
- `employee_schedules`
- `attendance_records`

### Schema 概念確定，但昨天正式名稱仍待核對

- 排班規則主檔（暫稱 `schedule_patterns`）
- 排班週期節點（暫稱 `schedule_pattern_days`）
- 員工排班規則套用（暫稱 `employee_schedule_assignments`）
- 班表調整／異動（暫稱 `employee_schedule_changes`）
- 打卡撤銷（暫稱 `attendance_cancellations`）
- 補登申請（暫稱 `attendance_correction_requests`）
- 補登審核（暫稱 `attendance_correction_approvals`）
- 出勤異常 Schema（正式名稱待恢復）

> 原討論明確否決 `employee_holiday_calendars`，因此本架構不建立該表。

下一批：加班／補休。