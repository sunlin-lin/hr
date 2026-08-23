# 人資系統需求原始對話紀錄 — Part 007

> 來源：ChatGPT 對話「整理人資系統需求」  
> 時區：Asia/Taipei  
> 範圍：2026/08/22 21:58:41 ～ 2026/08/22 22:09:32  
> 說明：保留原始對話順序，未去除重複、舊版或後續已修正內容。

---

## 2026/08/22 21:58:41

### 使用者

那如果你排好班了，中途條班呢

### 助理

可以，而且**中途調班是一定會發生的**。所以這也是我前面說 `employee_schedules` 不能只是單純「週期計算結果」的原因。

我會把它設計成：

> **原始排班 + 調班異動 + 最終班表**

而不是直接覆蓋原資料。

---

# 例如

原本 9 月班表已經發布：

```text
9/10 → A班 08:00～17:00
```

9/8 主管通知：

> 王小明 9/10 改上 B班 14:00～23:00

這就是**調班**。

---

# 我建議增加 `schedule_changes`

### 說明

**員工已排定班表的異動紀錄。**

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 調班紀錄 ID |
| `employee_schedule_id` | `uuid` | 原班表 ID |
| `employee_id` | `uuid` | 員工 ID |
| `original_shift_id` | `uuid` | 原班別 |
| `new_shift_id` | `uuid` | 新班別 |
| `original_schedule_day_type_code` | `integer` | 原日期性質 |
| `new_schedule_day_type_code` | `integer` | 新日期性質 |
| `original_scheduled_work_flag` | `boolean` | 原是否出勤 |
| `new_scheduled_work_flag` | `boolean` | 新是否出勤 |
| `reason` | `string` | 調班原因 |
| `status_code` | `integer` | 調班狀態 |
| `requested_by` | `uuid` | 申請人 |
| `approved_by` | `uuid` | 核准人 |
| `approved_at` | `datetime` | 核准時間 |
| `effective_at` | `datetime` | 生效時間 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 但是我不建議「調班後只存 changes」

最終的 `employee_schedules` 還是要變成：

```text
9/10 → B班 14:00～23:00
```

因為之後所有系統都應該看：

```text
employee_schedules
```

例如打卡：

```text
14:03 → 上班
23:02 → 下班
```

系統就知道：

```text
應上班 14:00
應下班 23:00
```

而不是每次都：

```text
employee_schedules
↓
schedule_changes
↓
重新計算
```

---

# 所以會是這種關係

```text
原始排班規則
      ↓
employee_schedules
      │
      │ 調班
      ▼
schedule_changes
      │
      │ 核准
      ▼
更新 employee_schedules
```

例如：

```text
原本：

employee_schedules
9/10 → A班


調班：

schedule_changes
A班 → B班
原因：支援夜班
核准：主管


最終：

employee_schedules
9/10 → B班
```

---

# 但還有一種更麻煩的「換班」

例如：

### 王小明

```text
9/10 → A班
```

### 李小華

```text
9/10 → B班
```

兩個人互換：

```text
王小明 → B班
李小華 → A班
```

這其實不是一筆調班，而是**一次換班操作包含兩個員工的班表異動**。

所以如果要做到完整，我會再把：

```text
schedule_changes
```

拆成：

```text
schedule_change_requests
        ↓
schedule_change_details
```

一個「調班事件」裡可以有很多筆員工變更。

例如：

```text
調班事件 #1001

王小明
A班 → B班

李小華
B班 → A班
```

---

# 我更推薦這個最終設計

```text
schedule_periods
        │
        ▼
employee_schedules
        │
        │
        └──── schedule_change_details
                     ▲
                     │
             schedule_changes
```

### `schedule_changes`

代表：

> **這次調班事件**

例如：

```text
#1001
王小明與李小華換班
原因：人力調度
申請人：主管
核准人：經理
```

### `schedule_change_details`

代表：

> **這次調班到底改了誰的哪一天**

```text
王小明
9/10
A → B

李小華
9/10
B → A
```

---

# 還有一個關鍵：已經打卡後不能隨便改

例如：

```text
9/10

原班：
08:00～17:00

實際：
08:03 上班
17:02 下班
```

這時主管想把：

```text
9/10
改成夜班
```

系統應該**不直接允許普通調班覆蓋**。

因為這會影響：

- 遲到
- 早退
- 工作時間
- 加班
- 薪資

這種應該變成：

> **已產生出勤資料的班表異動，需要特殊權限 / 異常處理。**

---

## 所以整個邏輯會非常漂亮

```text
【排班規則】
二休二 / 固定班 / 輪班
       ↓
【正式班表】
employee_schedules
       ↓
【中途調班】
schedule_changes
       ↓
【更新最終班表】
employee_schedules
       ↓
【員工打卡】
attendance_records
       ↓
【出勤判定】
attendance_results
       ↓
【加班】
overtime
       ↓
【薪資】
payroll
```

**最重要的是：`employee_schedules` 永遠代表「當下正式有效的班表」。**

而 `schedule_changes` 負責留下「它為什麼從 A 變成 B」的歷史。

這樣即使半年後查資料，也可以回答：

> 「王小明 9/10 為什麼原本是早班，最後變成晚班？」

而不是只剩下一個「晚班」給你看。

---

## 2026/08/22 21:59:01

### 使用者

那如果員工請假呢

### 助理

這裡要特別區分：

> **請假不是調班，也不應該直接修改 `employee_schedules`。**

因為「原本應該上班」和「後來請假沒來」是兩件不同的事。

例如：

```text
9/10
原班表：08:00～17:00
```

員工請一天假：

```text
9/10
請假：事假
```

最後應該是：

```text
班表：仍然是 08:00～17:00
出勤：未出勤
請假：事假
```

而不是把班表改成「休息」。

---

# 一、所以會多一個 `leave_requests`

### `leave_requests`

**說明：**

> 員工請假申請，記錄請假的原因、期間、類型及審核狀態。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 請假申請 ID |
| `company_id` | `uuid` | 公司 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `leave_type_code` | `integer` | 假別代碼 |
| `start_datetime` | `datetime` | 請假開始 |
| `end_datetime` | `datetime` | 請假結束 |
| `leave_minutes` | `integer` | 請假分鐘數 |
| `reason` | `string` | 請假原因 |
| `status_code` | `integer` | 申請狀態 |
| `applicant_id` | `uuid` | 申請人 |
| `approved_by` | `uuid` | 核准人 |
| `approved_at` | `datetime` | 核准時間 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 二、但是「一天請假」還不夠

例如員工：

```text
08:00～12:00 工作
12:00～13:00 休息
13:00～17:00 工作
```

他可能只請：

```text
13:00～17:00
```

所以 `leave_requests` 可以記錄申請區間。

但如果要做到很完整，我會再拆：

## `leave_request_details`

**說明：**

> 請假申請在各個工作日的實際請假明細。

例如：

```text
leave_request #1001

9/10
13:00～17:00
240分鐘
```

如果請跨日：

```text
9/10 → 4小時
9/11 → 8小時
9/12 → 4小時
```

就可以各自記錄。

---

# 三、為什麼這樣很重要？

因為你的班表是：

```text
employee_schedules
```

請假是：

```text
leave_requests
```

實際打卡是：

```text
attendance_records
```

三者各自代表不同事情。

例如：

```text
9/10
班表：A班 08:00～17:00
       ↓
員工請假：09:00～12:00
       ↓
實際打卡：13:00
```

系統最後可以判定：

```text
08:00～09:00
應出勤，但沒打卡 → 遲到？

09:00～12:00
已核准請假 → 不算缺勤

12:00～13:00
休息

13:00
正常上班
```

這就是為什麼**不能直接把班表改成休假**。

---

# 四、那如果請「整天假」呢？

例如：

```text
employee_schedules

9/10 → A班
```

請假：

```text
leave_requests

9/10
全天
特休
```

最後：

```text
班表
9/10 → A班

請假
9/10 → 特休 8小時

出勤
9/10 → 無需打卡
```

非常清楚。

---

# 五、那如果「請假」後又取消呢？

例如：

```text
9/10
原本 A班
```

員工：

```text
申請特休
```

主管核准。

後來員工：

> 我不請假了。

那我們**不要刪掉請假紀錄**。

而是：

```text
leave_requests.status_code

→ 已取消
```

然後恢復：

```text
employee_schedules
9/10 → A班
```

---

# 六、請假與加班也能自然分開

例如：

```text
9/10
08:00～17:00 A班
```

員工：

```text
08:00～12:00 請假
13:00～17:00 上班
17:00～20:00 加班
```

系統就可以同時存在：

```text
employee_schedules
        │
        ├── leave_requests
        │
        ├── attendance_records
        │
        └── overtime_requests
```

而不是互相修改資料。

---

# 七、所以現在整個人事出勤關係變成

```text
                    employee
                       │
                       ▼
              employee_schedules
                 /       |       \
                /        |        \
               ▼         ▼         ▼
          請假申請      打卡       加班申請
             │           │           │
             ▼           ▼           ▼
      leave_requests  attendance  overtime
                       records     requests
```

而最終：

```text
employee_schedules
= 應該怎麼上班

leave_requests
= 為什麼這段時間不用上班

attendance_records
= 實際怎麼打卡

overtime_requests
= 為什麼這段時間要算加班
```

我會非常堅持這四個概念分開。

尤其是**不要因為請假就把 `employee_schedules.schedule_type` 改成休假**，否則未來你查「這個人原本被排什麼班」時，資料就失真了。

---

## 2026/08/22 21:59:23

### 使用者

在幫我思考還有沒有其他狀況

### 助理

有，而且如果我們現在是在做一套 **SaaS、多公司、完整 HR + 打卡 + 排班 + 薪資系統**，目前的「班表 → 請假 → 打卡 → 加班」概念已經對了，但還有一些情境現在不處理，之後很容易補洞補到很痛苦。

我先**不寫程式、不急著定 Schema**，單純把需求情境盤出來。

---

# 1. 臨時不上班，但不是請假

例如：

> 今天公司臨時宣布停班。

或者：

> 台風天，公司宣布今天不用上班。

這跟員工請假完全不同。

```text
原本班表
8/22 → 工作

公司臨時通知
8/22 → 公司停班
```

這時：

- 員工沒有請假
- 班表原本也沒有錯
- 但今天實際上不用出勤

所以需要一種：

> **公司/部門層級的臨時出勤異動**

---

# 2. 員工臨時被叫來上班

這個跟「調班」不同。

例如：

```text
原本：
8/22 → 休息

臨時：
主管通知 → 8/22 18:00～22:00 回來上班
```

這時應該可以形成：

```text
原班表 → 休息
臨時出勤 → 18:00～22:00
```

而且可能涉及：

- 休息日加班
- 假日出勤
- 加班申請
- 加班費

所以不能單純把原本休息日改成工作日。

---

# 3. 換班

前面有提到，但我認為要特別列出。

```text
A員工
8/25 → A班

B員工
8/25 → B班
```

兩人互換：

```text
A → B
B → A
```

這不是單純「修改一個人的班表」。

而是一個：

> **換班事件**

可能還需要：

- 雙方同意
- 主管核准
- 系統檢查資格

例如 A 可以上 B 班，但 B 沒有資格上 A 班，就不能換。

---

# 4. 部門整批調班

這個 HR 系統很可能會遇到。

例如：

> 研發部 9/1 全部改成 09:00～18:00。

如果有 100 人：

```text
100 筆 employee_schedules
```

不能讓管理者一個一個改。

所以未來需要支援：

> **批次排班 / 批次調班**

---

# 5. 員工換部門

這個你之前已經特別定義：

> 員工與部門不一定永遠只有一個部門。

那就會產生一個很重要的問題：

```text
8/31
研發部

9/1
業務部
```

那 9/1 的班表應該套：

```text
研發部班表？
業務部班表？
```

因此班表規則未來可能不只：

```text
員工 → 班表
```

也可能：

```text
部門 → 班表規則
```

再由員工繼承。

這個要先想清楚。

---

# 6. 同一個員工不同日期不同班表

例如：

```text
週一 → A班
週二 → B班
週三 → C班
```

這其實就是輪班。

目前週期模型可以處理。

---

# 7. 同一天多段工作

例如：

```text
08:00～12:00
13:00～17:00
18:00～21:00
```

甚至：

```text
上午工作
下午休息
晚上再工作
```

所以班表不能只設：

```text
start_time
end_time
```

我們前面設 `shift_work_periods` 是正確的。

---

# 8. 跨日班

例如：

```text
22:00 → 06:00
```

尤其是：

- 保全
- 醫療
- 工廠
- 物流
- 餐飲

都可能遇到。

所以必須有：

```text
start_day_offset
end_day_offset
```

這個不能拿掉。

---

# 9. 跨日請假

例如：

```text
夜班
22:00～06:00
```

員工請假：

```text
23:00～02:00
```

這個請假日期到底算哪一天？

應該依照：

> **班次工作日期**

來判斷，而不是單純看 datetime 的 calendar date。

這個要在需求層先定義。

---

# 10. 忘記打卡

你前面已經定義：

> 可以補登。

但還有幾種：

### 忘記上班

```text
沒有上班卡
有下班卡
```

### 忘記下班

```text
有上班卡
沒有下班卡
```

### 上下班都忘記

```text
完全沒有打卡
```

這三種最好都可以處理。

---

# 11. 誤打卡

例如：

```text
08:00 上班
08:02 上班
```

或者：

```text
17:00 下班
17:01 下班
```

你之前已經說：

> 員工可以自己撤銷打卡。

那就會需要：

```text
原始打卡
↓
撤銷
↓
保留紀錄
```

**不能真的 DELETE。**

---

# 12. GPS 不一定有

你之前說：

> GPS 可以記錄，不強制。

那就要接受：

```text
GPS 有
GPS 沒有
```

例如：

```text
打卡機
Web
手機
```

來源不同。

所以打卡應該記錄：

```text
打卡來源
GPS
IP
裝置資訊
```

但 GPS 不是必要欄位。

---

# 13. 同一個人一天可能有很多次打卡

不能假設：

```text
一個人一天只有兩筆
```

例如：

```text
08:00 上班
12:00 外出
13:00 回來
17:00 下班
19:00 加班
22:00 下班
```

所以：

> **打卡紀錄一定是一對多。**

---

# 14. 外出 / 公出

這個我認為 HR 系統很容易遇到。

例如：

```text
今天 09:00～12:00
去客戶公司
```

他沒有在公司打卡。

但：

> 這不是請假。

可能是：

- 公出
- 出差
- 外勤

所以未來應該獨立處理。

---

# 15. 出差

甚至：

```text
8/22～8/24
台北出差
```

這時：

- 原本班表還是存在
- 不一定在公司 GPS
- 可能仍然要正常出勤
- 可能產生出差補助

所以不能把它當請假。

---

# 16. 彈性工時

例如：

```text
核心時間：
10:00～16:00

一天工作滿 8 小時即可
```

這跟：

```text
08:00～17:00
```

完全不同。

所以前面 `shift_definitions.is_flexible` 是有意義的。

---

# 17. 遲到寬限

例如：

```text
08:00 上班
08:05 前不算遲到
```

甚至不同班別不同：

```text
A班 → 5分鐘
B班 → 10分鐘
```

所以這應該屬於**班別規則**。

---

# 18. 打卡時間取整

有些公司可能規定：

```text
08:07
→ 08:00

08:08
→ 08:15
```

或者：

```text
以 15 分鐘為單位
```

這會直接影響：

- 遲到
- 工時
- 加班

所以前面 `rounding_rule_code` 保留是合理的。

---

# 19. 加班不一定是「下班後」

例如：

```text
正常班
08:00～17:00
```

員工：

```text
06:00～08:00
```

也可能是加班。

或者：

```text
午休時間工作
```

所以不要把加班設計成：

> 下班時間之後才叫加班。

---

# 20. 請假不一定整天

這個我們剛剛已經碰到。

可能：

```text
上午請假
下午上班
```

甚至：

```text
請假 2 小時
```

所以請假必須支援**分鐘級**。

---

# 21. 請假可能跨多天

例如：

```text
8/25～8/29
特休
```

但每一天的應出勤時間可能不同。

因此不能單純：

```text
5天 × 8小時
```

而應該依每天班表計算。

---

# 22. 請假與班表衝突

例如：

```text
員工原本 8/25 休息
```

卻申請：

```text
8/25 特休
```

系統應該怎麼處理？

我會建議：

> **原則上不應該產生有效請假時數。**

因為他本來就不用上班。

這是很重要的驗證規則。

---

# 23. 已請假後調班

例如：

```text
8/25
原本 A班
員工已請假
```

後來主管：

```text
把 8/25 改成 B班
```

這時會產生：

> 原本請假的內容是否仍有效？

這就需要系統重新檢查：

```text
請假
↔
新班表
```

所以班表變更不能只是「改時間」。

---

# 24. 已打卡後請假

例如：

```text
08:00 上班
```

下午突然請假：

```text
14:00～17:00
```

這是非常正常的情況。

所以：

```text
attendance_records
+
leave_requests
```

可以同時存在。

最後結果：

```text
08:00～14:00 → 出勤
14:00～17:00 → 核准請假
```

---

# 25. 已加班後才請假 / 修改班表

這就更複雜：

```text
正常班
08:00～17:00

加班申請
17:00～20:00
```

後來班表被修改成：

```text
10:00～19:00
```

原本的加班資格可能完全不同。

所以：

> **班表修改後，需要重新驗證已存在的請假與加班申請。**

---

# 26. 最重要的一個：資料「版本」

我現在反而認為這是我們下一階段最需要處理的。

因為：

```text
班表
↓
請假
↓
打卡
↓
加班
↓
薪資
```

都是互相影響的。

所以不能只想：

```text
employee_schedules
```

而要有：

> **「當時系統認定的班表」**

例如：

```text
9/10
原本 A班
```

後來改：

```text
B班
```

但員工已經打卡。

那 9/10 的出勤判定，到底依 A 還是 B？

這時就需要**班表異動歷史 / 版本概念**。

---

# 我目前會把整個出勤領域分成 6 層

```text
① 班別
   ↓
shift_definitions
shift_work_periods
shift_breaks
shift_rules

② 排班規則
   ↓
schedule_cycles
schedule_cycle_details

③ 實際班表
   ↓
employee_schedules
schedule_periods
schedule_changes

④ 員工事件
   ↓
leave_requests
出差
公出
臨時出勤
...

⑤ 實際行為
   ↓
attendance_records

⑥ 系統判定
   ↓
attendance_results
overtime_requests
```

這樣會比把所有東西塞進 `employee_schedules` 健康很多。

---

## 我目前最想先確定的 5 個需求

在正式定 Schema 前，我建議我們先把這五個規則定死：

**A. 臨時不上班 / 公司停班**

**B. 臨時叫員工回來上班**

**C. 換班**

**D. 公出 / 外勤 / 出差**

**E. 班表發布後，已經有人打卡時，還能不能調班**

尤其 **E** 最重要。

因為這會決定我們到底只需要「調班紀錄」，還是要做完整的**班表版本 / 生效時間 / 異動鎖定機制**。

---

## 2026/08/22 22:01:51

### 使用者

同意你的5個需求
把Schema   定義出來，跟設計理念

### 助理

可以。既然你同意這 5 個需求，我會把目前的**排班／出勤相關 Schema 正式整理成一套完整設計**。

這一版我會遵守我們前面已經確定的原則：

- 不使用 `enum`
- 狀態、類型使用 `integer code`
- `code` 的意義寫在欄位註釋／文件中
- 班表與請假、打卡、加班分離
- `employee_schedules` 是「最終有效班表」
- 週期不以 7 天為限制
- 支援二休二、四休二、三班輪班
- 支援零工、兼職、臨時排班
- 支援跨日班
- 支援中途調班
- 支援換班
- 支援公司停班
- 支援臨時叫班
- 支援公出、外勤、出差
- 已產生打卡後，班表異動必須留下歷史
- 不直接刪除歷史資料

---

# 一、先確立整體設計理念

整套出勤系統，我建議用這個核心概念：

```text
「規則」→「排定」→「異動」→「實際」→「判定」
```

也就是：

```text
班別規則
    ↓
排班規則
    ↓
實際班表
    ↓
請假 / 調班 / 公出 / 臨時出勤
    ↓
實際打卡
    ↓
出勤結果
    ↓
加班 / 薪資
```

最重要的是：

> **任何一層都不要竄改上一層的原始意義。**

例如：

員工請假，不修改班表。

員工打卡，不修改班表。

主管調班，不修改原本的週期規則。

---

# 二、班別 Schema

## 1. `shift_definitions`

### 設計理念

定義「這個班是什麼」。

例如：

```text
一般班
早班
晚班
夜班
彈性班
```

它不是員工的班表。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 班別 ID |
| `company_id` | `uuid` | 所屬公司 |
| `code` | `varchar` | 班別代碼 |
| `name` | `varchar` | 班別名稱 |
| `work_type_code` | `integer` | 工時類型代碼 |
| `is_overnight` | `boolean` | 是否跨日 |
| `is_flexible` | `boolean` | 是否彈性工時 |
| `required_work_minutes` | `integer` | 應工作分鐘 |
| `description` | `text` | 說明 |
| `is_active` | `boolean` | 是否啟用 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 三、工作時段

## 2. `shift_work_periods`

### 設計理念

**一個班別可以有多個工作時段。**

不要把：

```text
start_time
end_time
```

直接塞進 `shift_definitions`。

因為可能：

```text
08:00～12:00
13:00～17:00
18:00～21:00
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 時段 ID |
| `shift_id` | `uuid` | 班別 ID |
| `sequence` | `integer` | 時段順序 |
| `start_time` | `time` | 開始時間 |
| `end_time` | `time` | 結束時間 |
| `start_day_offset` | `integer` | 開始日偏移 |
| `end_day_offset` | `integer` | 結束日偏移 |
| `required_work_minutes` | `integer` | 應工作分鐘 |
| `late_grace_minutes` | `integer` | 遲到寬限 |
| `early_leave_grace_minutes` | `integer` | 早退寬限 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 四、休息時間

## 3. `shift_breaks`

### 設計理念

休息時間與工作時段分開。

因為休息可能：

- 有薪
- 無薪
- 多段
- 跨日

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 休息 ID |
| `shift_id` | `uuid` | 班別 ID |
| `sequence` | `integer` | 順序 |
| `start_time` | `time` | 開始時間 |
| `end_time` | `time` | 結束時間 |
| `start_day_offset` | `integer` | 開始日偏移 |
| `end_day_offset` | `integer` | 結束日偏移 |
| `break_type_code` | `integer` | 休息類型 |
| `is_paid` | `boolean` | 是否計薪 |
| `description` | `text` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 五、班別規則

## 4. `shift_rules`

### 設計理念

把「時間」和「怎麼判定」分開。

例如：

```text
08:00 上班
08:05 以前不算遲到
```

這是規則，不是時間本身。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 規則 ID |
| `shift_id` | `uuid` | 班別 ID |
| `late_rule_code` | `integer` | 遲到規則 |
| `early_leave_rule_code` | `integer` | 早退規則 |
| `overtime_rule_code` | `integer` | 加班規則 |
| `rounding_rule_code` | `integer` | 時間取整規則 |
| `minimum_work_minutes` | `integer` | 最低工作分鐘 |
| `maximum_work_minutes` | `integer` | 最大工作分鐘 |
| `description` | `text` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 六、班表週期

## 5. `schedule_cycles`

### 設計理念

**週期不是一週。**

這是這次設計最重要的修正。

支援：

```text
7 天
4 天 → 二休二
6 天 → 四休二
12 天 → 三班輪班
28 天
```

甚至任意週期。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 週期 ID |
| `company_id` | `uuid` | 公司 ID |
| `code` | `varchar` | 週期代碼 |
| `name` | `varchar` | 週期名稱 |
| `cycle_length` | `integer` | 週期天數 |
| `description` | `text` | 說明 |
| `is_active` | `boolean` | 是否啟用 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |
| `deleted_at` | `datetime` | 軟刪除 |

---

# 七、週期明細

## 6. `schedule_cycle_details`

### 設計理念

定義：

> 週期中的第 N 天要做什麼。

例如二休二：

```text
Day 1 → 工作
Day 2 → 工作
Day 3 → 休息
Day 4 → 休息
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 明細 ID |
| `schedule_cycle_id` | `uuid` | 週期 ID |
| `cycle_day` | `integer` | 週期第幾天 |
| `shift_id` | `uuid` | 班別 ID，可 NULL |
| `schedule_day_type_code` | `integer` | 日期性質 |
| `scheduled_work_flag` | `boolean` | 是否排定出勤 |
| `overtime_eligibility_code` | `integer` | 加班申請資格 |
| `description` | `text` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

這裡就是你剛才提出的：

> **班表體系決定這一天是否具備加班資格。**

---

# 八、員工套用週期

## 7. `employee_schedule_cycles`

### 設計理念

表示：

> 這個員工從哪一天開始套用哪個週期。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 |
| `schedule_cycle_id` | `uuid` | 週期 ID |
| `start_date` | `date` | 開始日期 |
| `end_date` | `date` | 結束日期 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 九、排班期間

## 8. `schedule_periods`

### 設計理念

解決：

> 「這次到底排了多久？」

例如：

```text
2026/09/01 ～ 2026/09/30
```

是一個排班期間。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 排班期間 ID |
| `company_id` | `uuid` | 公司 ID |
| `name` | `varchar` | 名稱 |
| `start_date` | `date` | 開始日期 |
| `end_date` | `date` | 結束日期 |
| `status_code` | `integer` | 排班期間狀態 |
| `published_at` | `datetime` | 發布時間 |
| `published_by` | `uuid` | 發布人員 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 十、正式員工班表

## 9. `employee_schedules`

### 設計理念

這張是整個排班系統的核心。

> **代表員工在某一天「最終被安排」的班表。**

它不是週期規則，也不是打卡。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 班表 ID |
| `schedule_period_id` | `uuid` | 排班期間 |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 |
| `work_date` | `date` | 工作日期 |
| `shift_id` | `uuid` | 班別，可 NULL |
| `schedule_day_type_code` | `integer` | 日期性質 |
| `scheduled_work_flag` | `boolean` | 是否安排出勤 |
| `overtime_eligibility_code` | `integer` | 加班資格 |
| `source_type_code` | `integer` | 班表來源 |
| `source_id` | `uuid` | 來源 ID |
| `note` | `text` | 備註 |
| `created_by` | `uuid` | 建立人員 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 十一、調班

這裡我會正式採用「事件 + 明細」兩層。

## 10. `schedule_changes`

### 設計理念

代表：

> **一次調班操作。**

例如：

```text
王小明與李小華換班
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 調班事件 ID |
| `company_id` | `uuid` | 公司 ID |
| `schedule_period_id` | `uuid` | 排班期間 |
| `change_type_code` | `integer` | 調班類型 |
| `reason` | `text` | 原因 |
| `status_code` | `integer` | 狀態 |
| `requested_by` | `uuid` | 申請人 |
| `approved_by` | `uuid` | 核准人 |
| `approved_at` | `datetime` | 核准時間 |
| `effective_at` | `datetime` | 生效時間 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

## 11. `schedule_change_details`

### 設計理念

代表：

> 這次調班實際改了誰、哪一天、從什麼變成什麼。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 明細 ID |
| `schedule_change_id` | `uuid` | 調班事件 |
| `employee_schedule_id` | `uuid` | 原班表 |
| `employee_id` | `uuid` | 員工 |
| `work_date` | `date` | 日期 |
| `original_shift_id` | `uuid` | 原班別 |
| `new_shift_id` | `uuid` | 新班別 |
| `original_day_type_code` | `integer` | 原日期性質 |
| `new_day_type_code` | `integer` | 新日期性質 |
| `original_work_flag` | `boolean` | 原是否出勤 |
| `new_work_flag` | `boolean` | 新是否出勤 |
| `original_overtime_eligibility_code` | `integer` | 原加班資格 |
| `new_overtime_eligibility_code` | `integer` | 新加班資格 |
| `created_at` | `datetime` | 建立時間 |

---

# 十二、公司臨時停班

這是前面同意的第一個特殊情境。

## 12. `schedule_exceptions`

### 設計理念

處理：

> **原本班表沒有錯，但公司臨時改變出勤規則。**

例如：

```text
颱風停班
公司臨時休假
部門臨時停工
特殊事件
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 異常事件 ID |
| `company_id` | `uuid` | 公司 ID |
| `scope_type_code` | `integer` | 適用範圍 |
| `department_id` | `uuid` | 部門，可 NULL |
| `employee_id` | `uuid` | 員工，可 NULL |
| `start_datetime` | `datetime` | 開始 |
| `end_datetime` | `datetime` | 結束 |
| `exception_type_code` | `integer` | 異動類型 |
| `reason` | `text` | 原因 |
| `status_code` | `integer` | 狀態 |
| `created_by` | `uuid` | 建立人員 |
| `approved_by` | `uuid` | 核准人 |
| `approved_at` | `datetime` | 核准時間 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 十三、臨時叫班

第二種特殊情境：

> 原本沒排班，但臨時叫他來。

我不建議直接修改原本的休息日。

## 13. `temporary_work_assignments`

### 設計理念

記錄：

> 原本沒有排班，後來臨時要求員工出勤。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 臨時出勤 ID |
| `company_id` | `uuid` | 公司 |
| `employee_id` | `uuid` | 員工 |
| `employment_id` | `uuid` | 任職 |
| `work_date` | `date` | 日期 |
| `shift_id` | `uuid` | 班別 |
| `start_datetime` | `datetime` | 預計開始 |
| `end_datetime` | `datetime` | 預計結束 |
| `work_type_code` | `integer` | 出勤類型 |
| `overtime_eligibility_code` | `integer` | 加班資格 |
| `reason` | `text` | 原因 |
| `status_code` | `integer` | 狀態 |
| `created_by` | `uuid` | 建立人員 |
| `approved_by` | `uuid` | 核准人 |
| `approved_at` | `datetime` | 核准時間 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

這樣：

```text
原本：
8/22 → 休息

臨時叫班：
18:00～22:00
```

兩件事情都保留。

---

# 十四、請假

## 14. `leave_requests`

### 設計理念

> **請假不修改班表。**

班表表示「應該怎麼工作」。

請假表示：

> 「原本應該工作，但經核准後不用工作。」

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 請假 ID |
| `company_id` | `uuid` | 公司 |
| `employee_id` | `uuid` | 員工 |
| `employment_id` | `uuid` | 任職 |
| `leave_type_code` | `integer` | 假別 |
| `start_datetime` | `datetime` | 開始 |
| `end_datetime` | `datetime` | 結束 |
| `leave_minutes` | `integer` | 請假分鐘 |
| `reason` | `text` | 原因 |
| `status_code` | `integer` | 狀態 |
| `applicant_id` | `uuid` | 申請人 |
| `approved_by` | `uuid` | 核准人 |
| `approved_at` | `datetime` | 核准時間 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 十五、請假每日明細

## 15. `leave_request_details`

### 設計理念

跨日請假時，每天的應出勤時間可能不同。

因此需要拆每日明細。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 明細 ID |
| `leave_request_id` | `uuid` | 請假申請 |
| `employee_schedule_id` | `uuid` | 對應班表 |
| `work_date` | `date` | 工作日期 |
| `start_datetime` | `datetime` | 請假開始 |
| `end_datetime` | `datetime` | 請假結束 |
| `leave_minutes` | `integer` | 請假分鐘 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 十六、公出 / 外勤 / 出差

我會把這三種先歸到一個概念：

> **非固定工作地點的工作活動。**

## 16. `work_assignments`

### 設計理念

員工仍然是在「工作」，只是沒有在原本工作地點。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 工作派遣 ID |
| `company_id` | `uuid` | 公司 |
| `employee_id` | `uuid` | 員工 |
| `employment_id` | `uuid` | 任職 |
| `assignment_type_code` | `integer` | 公出/外勤/出差 |
| `start_datetime` | `datetime` | 開始 |
| `end_datetime` | `datetime` | 結束 |
| `destination` | `string` | 地點 |
| `reason` | `text` | 原因 |
| `status_code` | `integer` | 狀態 |
| `approved_by` | `uuid` | 核准人 |
| `approved_at` | `datetime` | 核准時間 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 十七、這樣「公出」就不會被誤判成缺卡

例如：

```text
08:00
公司打卡

09:00～12:00
客戶公司

13:00
回公司
```

即使 GPS 顯示：

```text
09:30
不在公司
```

系統也知道：

```text
work_assignments
→ 公出
```

所以不應該直接判定異常。

---

# 十八、我會把「打卡」完全獨立

## 17. `attendance_records`

### 設計理念

只記錄：

> **員工實際做了什麼打卡動作。**

不要在這裡直接存「遲到」。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 打卡 ID |
| `company_id` | `uuid` | 公司 |
| `employee_id` | `uuid` | 員工 |
| `employment_id` | `uuid` | 任職 |
| `employee_schedule_id` | `uuid` | 對應班表 |
| `attendance_type_code` | `integer` | 上班/下班等 |
| `recorded_at` | `datetime` | 實際打卡時間 |
| `source_type_code` | `integer` | 打卡來源 |
| `latitude` | `decimal` | GPS 緯度，可 NULL |
| `longitude` | `decimal` | GPS 經度，可 NULL |
| `accuracy` | `decimal` | GPS 精度，可 NULL |
| `ip_address` | `string` | IP，可 NULL |
| `device_identifier` | `string` | 裝置識別，可 NULL |
| `status_code` | `integer` | 打卡狀態 |
| `cancelled_at` | `datetime` | 撤銷時間 |
| `cancelled_by` | `uuid` | 撤銷人 |
| `cancel_reason` | `text` | 撤銷原因 |
| `created_at` | `datetime` | 建立時間 |

**這張表原則上不 DELETE。**

---

# 十九、補卡

## 18. `attendance_correction_requests`

### 設計理念

員工忘記打卡，不直接幫他插一筆假的打卡紀錄。

而是：

```text
申請補卡
↓
主管核准
↓
產生有效出勤資料
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 補卡申請 ID |
| `company_id` | `uuid` | 公司 |
| `employee_id` | `uuid` | 員工 |
| `employment_id` | `uuid` | 任職 |
| `employee_schedule_id` | `uuid` | 對應班表 |
| `attendance_type_code` | `integer` | 上班/下班 |
| `requested_at` | `datetime` | 申請補卡時間 |
| `reason` | `text` | 補卡原因 |
| `status_code` | `integer` | 狀態 |
| `approved_by` | `uuid` | 核准人 |
| `approved_at` | `datetime` | 核准時間 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 二十、出勤結果

## 19. `attendance_results`

### 設計理念

這張是：

> **系統計算後的結果。**

例如：

```text
應上班 08:00
實際 08:07

→ 遲到 7 分鐘
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 出勤結果 ID |
| `employee_schedule_id` | `uuid` | 班表 |
| `employee_id` | `uuid` | 員工 |
| `work_date` | `date` | 日期 |
| `scheduled_minutes` | `integer` | 應工作分鐘 |
| `worked_minutes` | `integer` | 實際工作分鐘 |
| `late_minutes` | `integer` | 遲到 |
| `early_leave_minutes` | `integer` | 早退 |
| `absence_minutes` | `integer` | 缺勤 |
| `leave_minutes` | `integer` | 請假 |
| `overtime_minutes` | `integer` | 加班 |
| `result_status_code` | `integer` | 出勤結果 |
| `calculated_at` | `datetime` | 計算時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 二十一、最後是加班

這裡就可以非常乾淨。

## `overtime_requests`

### 核心邏輯

```text
employee_schedule
        ↓
是否具備 overtime_eligibility
        ↓
員工申請
        ↓
主管核准
        ↓
實際打卡
        ↓
實際加班
        ↓
薪資
```

所以**不是員工自己說要加班就可以算**。

---

# 二十二、整套關係

目前我會定義成：

```text
                        shift_definitions
                              │
                   ┌──────────┴──────────┐
                   ▼                     ▼
          shift_work_periods       shift_breaks
                   │
                   ▼
               shift_rules


                  排班規則
                     │
                     ▼
              schedule_cycles
                     │
                     ▼
          schedule_cycle_details
                     │
                     ▼
       employee_schedule_cycles
                     │
                     ▼
              schedule_periods
                     │
                     ▼
          employee_schedules
              │      │      │
              │      │      └── schedule_changes
              │      │
              │      └───────── leave_requests
              │
              ├──────────────── work_assignments
              │
              └──────────────── temporary_work_assignments
                     │
                     ▼
              attendance_records
                     │
                     ▼
              attendance_results
                     │
                     ▼
             overtime_requests
```

---

# 二十三、五個特殊情境現在都有位置

| 情境 | Schema |
|---|---|
| 公司臨時停班 | `schedule_exceptions` |
| 臨時叫員工上班 | `temporary_work_assignments` |
| 員工換班 / 調班 | `schedule_changes` + `schedule_change_details` |
| 公出 / 外勤 / 出差 | `work_assignments` |
| 已打卡後調班 | `schedule_changes` + 歷史資料 |

---

# 二十四、目前最重要的設計理念

我會把這套系統的核心定義成：

### `employee_schedules`

> **「應該怎麼上班」**

### `leave_requests`

> **「原本應該上班，但核准不用上班」**

### `temporary_work_assignments`

> **「原本不用上班，但臨時要求上班」**

### `work_assignments`

> **「有上班，但不是在原本工作地點」**

### `attendance_records`

> **「實際怎麼打卡」**

### `attendance_results`

> **「系統判定實際出勤結果」**

### `overtime_requests`

> **「員工申請加班，以及是否核准」**

### `schedule_changes`

> **「為什麼原本的班表後來變了」**

這樣資料的責任非常清楚，**不會出現一張 `employee_schedules` 什麼都在改的問題。**

另外，我會把「公司停班」再跟「員工個人臨時不用出勤」區分開，避免 `schedule_exceptions` 最後變成萬用垃圾桶。這部分等我們下一步檢查 Schema 時，可以再決定是否拆成公司/部門/員工三種範圍。

---

## 2026/08/22 22:04:42

### 使用者

我這邊有個補休的概念，加班不一定會被換成錢，也可以換成補休，你覺得適合加入這一次?

### 助理

我認為**適合現在就加入「補休」的概念**，但我不建議現在就把「補休怎麼折算、怎麼扣薪資」全部做進去。

原因很簡單：

> **補休其實是「加班產生的可用額度」，而不是單純的請假。**

如果現在不設計，之後做到薪資、請假時會很容易打掉重做。

---

# 一、先把概念分清楚

加班之後可能有兩種處理方式：

```text
加班
 │
 ├──→ 加班費
 │
 └──→ 補休
```

例如：

```text
加班 4 小時
```

可以：

```text
方案 A
→ 薪資計算 4 小時加班費

方案 B
→ 取得 4 小時補休額度
```

所以**補休不是假別本身的來源**。

它是由加班產生的。

---

# 二、我建議現在加入「補休額度」

可以先建立：

## `compensatory_leave_balances`

### 設計理念

記錄員工目前因加班而取得的補休額度。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 補休額度 ID |
| `company_id` | `uuid` | 公司 |
| `employee_id` | `uuid` | 員工 |
| `employment_id` | `uuid` | 任職 |
| `source_overtime_id` | `uuid` | 產生補休的加班紀錄 |
| `earned_minutes` | `integer` | 取得補休分鐘 |
| `used_minutes` | `integer` | 已使用分鐘 |
| `remaining_minutes` | `integer` | 剩餘分鐘 |
| `earned_at` | `datetime` | 取得時間 |
| `expire_at` | `datetime` | 到期時間，可 NULL |
| `status_code` | `integer` | 狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

---

# 三、但我會再改一下命名

其實我更推薦不要把它叫：

```text
compensatory_leave_balances
```

而是做成：

```text
compensatory_leave_credits
```

因為它本質上比較像：

> **補休額度**

例如：

```text
加班 #1001
產生 240 分鐘補休
```

產生：

```text
compensatory_leave_credits

240分鐘
```

員工後來請補休：

```text
使用 120分鐘
```

剩：

```text
120分鐘
```

---

# 四、補休使用時，不要直接修改額度

這一點非常重要。

不能：

```text
earned_minutes = 240
used_minutes = 120
```

然後把原資料一直改。

我會建議增加：

## `compensatory_leave_transactions`

### 設計理念

> **記錄補休額度的所有增加、使用、取消、到期等異動。**

例如：

```text id="0xv7iq"
+240
加班轉補休

-120
申請補休

+120
取消補休

-120
補休到期
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 異動 ID |
| `company_id` | `uuid` | 公司 |
| `employee_id` | `uuid` | 員工 |
| `employment_id` | `uuid` | 任職 |
| `credit_id` | `uuid` | 補休額度 |
| `transaction_type_code` | `integer` | 異動類型 |
| `minutes` | `integer` | 異動分鐘 |
| `reference_type_code` | `integer` | 關聯來源 |
| `reference_id` | `uuid` | 關聯資料 |
| `reason` | `text` | 原因 |
| `created_by` | `uuid` | 操作人 |
| `created_at` | `datetime` | 建立時間 |

這樣以後查：

> 「這個人的 8 小時補休是怎麼來的？」

可以完整追溯。

---

# 五、跟請假怎麼接？

這就是最重要的地方。

我們前面的：

```text
leave_requests
```

其實可以支援：

```text
leave_type_code
```

其中包含：

```text
補休
```

但是：

> **補休不是普通假別，它需要消耗 `compensatory_leave_credits`。**

例如：

```text
加班
↓
轉補休
↓
產生 240 分鐘補休額度
↓
員工申請補休
↓
leave_request
↓
扣除 120 分鐘
```

關係：

```text
overtime
   ↓
compensatory_leave_credit
   ↓
leave_request
   ↓
compensatory_leave_transaction
```

---

# 六、這也解決「加班到底拿錢還是補休」

我會在加班那邊增加：

```text
overtime_settlement_type_code
```

例如：

```text
1 = 加班費
2 = 補休
```

但我不建議現在把實際代碼定死，先把 Schema 定下來即可。

---

# 七、有一個情境現在就要想

例如：

```text
加班 4 小時
↓
選擇補休
↓
取得 4 小時補休
```

後來公司發現：

> 這筆加班不應該核准。

那怎麼辦？

所以我們必須保留：

```text
加班
↓
補休來源
↓
補休額度
```

不能只留下：

```text
員工目前有 4 小時補休
```

否則無法追溯。

---

# 八、還有「補休用掉一部分」

例如：

```text
補休額度：8 小時
```

第一次：

```text
請 2 小時
```

剩：

```text
6 小時
```

第二次：

```text
請 4 小時
```

剩：

```text
2 小時
```

第三次：

```text
請 2 小時
```

剛好用完。

所以：

> **補休一定要支援部分使用。**

因此 `minutes` 我會堅持使用 `integer`，以分鐘為單位。

---

# 九、還有「補休到期」

這個我建議現在就預留。

例如：

```text
取得日期
2026/08/01

到期日期
2027/08/01
```

到期後：

```text
剩餘 120 分鐘
↓
到期
↓
不可再使用
```

所以：

```text
expire_at
```

我會保留。

但**到期規則現在不用決定**，因為這涉及公司政策與適用法規。

---

# 十、所以我建議現在加入

目前出勤相關 Schema 增加：

```text
overtime_requests
       │
       │ 選擇補休
       ▼
compensatory_leave_credits
       │
       ├── compensatory_leave_transactions
       │
       └── leave_requests
```

核心概念：

```text
加班
 │
 ├──→ 加班費
 │
 └──→ 補休額度
          │
          ↓
       補休申請
          │
          ↓
       消耗額度
```

### 我的結論

**現在加入「補休」非常適合。**

但這一次只把**「加班可以轉換成補休、補休可以累積、使用、取消、到期、追溯來源」的資料模型定好**。

至於：

- 加班 1 小時換多少補休
- 不同加班倍率怎麼換
- 補休到期規則
- 補休未休完怎麼處理
- 補休是否可以折現
- 薪資結算時怎麼處理

這些我建議留到**薪資／加班規則**階段再定。

這樣目前的出勤 Schema 不會被薪資邏輯綁死。

---

## 2026/08/22 22:06:06

### 使用者

補休如果時間到沒有用完，還是要轉成薪資，加入這條件

### 助理

可以，這條我建議**現在就正式納入設計**，而且會影響補休的資料模型。

核心規則變成：

> **加班 → 選擇加班費或補休 → 補休在有效期限內使用 → 到期未使用的剩餘補休，轉為薪資結算。**

---

# 一、補休完整生命週期

```text
加班
 │
 ├──────────────→ 加班費
 │
 └──────────────→ 補休
                       │
                       ▼
                補休額度產生
                       │
              ┌────────┴────────┐
              ▼                 ▼
           使用補休          到期未使用
              │                 │
              ▼                 ▼
          剩餘額度           轉薪資
                                │
                                ▼
                           薪資結算
```

---

# 二、`compensatory_leave_credits` 要調整

原本的 Schema 我會增加「到期處理方式」。

### `compensatory_leave_credits`

**說明：**

> 記錄一筆由加班產生的補休額度，以及該額度的有效期限與到期處理方式。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 補休額度 ID |
| `company_id` | `uuid` | 公司 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 |
| `source_overtime_id` | `uuid` | 產生此補休的加班紀錄 |
| `earned_minutes` | `integer` | 取得補休分鐘數 |
| `used_minutes` | `integer` | 已使用分鐘數 |
| `remaining_minutes` | `integer` | 剩餘分鐘數 |
| `earned_at` | `datetime` | 取得時間 |
| `expire_at` | `datetime` | 補休到期時間 |
| `expiry_action_code` | `integer` | 到期處理方式 |
| `expired_at` | `datetime` | 實際到期處理時間 |
| `settled_minutes` | `integer` | 已轉薪資的分鐘數 |
| `status_code` | `integer` | 補休狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 更新時間 |

其中：

```text
expiry_action_code
```

目前可以定義：

```text
1 = 轉薪資
```

但我們不需要現在把所有代碼塞死。

---

# 三、為什麼要 `settled_minutes`

這個很重要。

例如：

```text
取得補休：8小時
```

員工使用：

```text
4小時
```

到期：

```text
剩餘4小時
```

這 4 小時：

```text
→ 轉薪資
```

所以最後：

```text
earned_minutes   = 480
used_minutes     = 240
remaining_minutes = 240
settled_minutes  = 240
```

但這裡有一個問題：

**`remaining_minutes` 到期後應該變成 0。**

所以其實我會更推薦：

```text
earned_minutes
used_minutes
settled_minutes
remaining_minutes
```

其中：

```text
remaining_minutes
=
earned_minutes
-
used_minutes
-
settled_minutes
```

到期結算後：

```text
480 - 240 - 240 = 0
```

---

# 四、但是不能只有這張表

我們之前決定要有：

### `compensatory_leave_transactions`

這張現在更重要。

例如：

```text
+480
加班轉補休

-240
員工使用補休

-240
補休到期轉薪資
```

完整歷史：

```text
2026/08/01
+480
加班 #1001 轉補休

2026/08/15
-240
補休申請 #2001

2027/08/01
-240
補休到期轉薪資
```

這樣未來薪資系統才能知道：

> **為什麼這個月突然多了一筆補休轉薪資？**

---

# 五、所以 Transaction 增加一種用途

### `compensatory_leave_transactions`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 異動 ID |
| `company_id` | `uuid` | 公司 |
| `employee_id` | `uuid` | 員工 |
| `employment_id` | `uuid` | 任職 |
| `credit_id` | `uuid` | 補休額度 |
| `transaction_type_code` | `integer` | 異動類型 |
| `minutes` | `integer` | 異動分鐘 |
| `reference_type_code` | `integer` | 關聯資料類型 |
| `reference_id` | `uuid` | 關聯資料 ID |
| `reason` | `text` | 原因 |
| `created_by` | `uuid` | 操作人 |
| `created_at` | `datetime` | 建立時間 |

其中 `transaction_type_code` 可以包含：

```text
補休取得
補休使用
補休取消返還
補休到期轉薪資
補休人工調整
```

實際代碼之後再整理。

---

# 六、薪資系統怎麼接？

這裡我反而建議：

**不要在補休 Schema 裡直接存薪資金額。**

因為：

```text
補休 4 小時
```

不一定等於：

```text
4 × 某個固定時薪
```

實際金額應該由薪資系統依當期規則計算。

所以：

```text
補休到期
    ↓
產生「轉薪資」事件
    ↓
薪資系統取得
settled_minutes
    ↓
依薪資規則計算金額
```

這樣出勤系統跟薪資系統不會耦合。

---

# 七、甚至可以留下薪資來源

未來薪資結算後，可以讓：

```text
compensatory_leave_transactions
```

有：

```text
reference_type_code
reference_id
```

例如：

```text
reference_type_code = 薪資結算
reference_id = payroll_item_id
```

就可以追：

```text
補休
↓
到期
↓
轉薪資
↓
2027/08 薪資
↓
補休轉薪資 4 小時
↓
金額 NT$xxxx
```

這個追溯鏈非常完整。

---

# 八、這也產生一個很重要的規則

如果補休：

```text
2027/08/01 到期
```

那麼：

> **到期當下不能再申請補休。**

而且系統應該在到期時自動：

```text
remaining_minutes
↓
settled_minutes
```

並建立：

```text
compensatory_leave_transaction
```

---

# 九、我會把規則正式定成

### 補休規則

1. 加班核准後，可選擇轉換為補休。
2. 補休以「分鐘」為最小單位。
3. 每筆補休都有自己的來源加班紀錄。
4. 每筆補休可以有自己的有效期限。
5. 補休可以分次使用。
6. 使用補休會產生交易紀錄。
7. 取消補休會返還原額度。
8. 補休到期後，剩餘額度**不得直接消失**。
9. 到期剩餘額度必須轉入薪資結算。
10. 補休轉薪資必須留下交易紀錄。
11. 補休本身不直接保存最終薪資金額。
12. 薪資金額由薪資系統依當期薪資規則計算。

---

## 最後整個關係就變成

```text
overtime_requests
       │
       │ 轉補休
       ▼
compensatory_leave_credits
       │
       ├──────────────┐
       ▼              ▼
leave_requests     到期
       │              │
       ▼              ▼
使用補休          轉薪資
       │              │
       └──────┬───────┘
              ▼
compensatory_leave_transactions
              │
              ▼
           Payroll
```

我認為這條規則**現在加入是正確的**。而且我們之後做薪資時，只要接上「補休到期轉薪資」這個來源，就不用回頭修改整個出勤架構。

---

## 2026/08/22 22:06:53

### 使用者

你薪資是事後算，還是當下直接算?

### 助理

如果是我們現在這套 HR SaaS，我會建議：

> **薪資不要在「事件發生當下」直接算錢，而是事後進行薪資結算。**
>
> 但「當下」要把**薪資計算所需的事件與資料鎖定、保存**。

這兩件事情要分開。

### 例如加班

員工 8/10 加班 3 小時：

```text
8/10
打卡
 ↓
出勤判定
 ↓
加班 3 小時
 ↓
選擇「補休」
 ↓
產生 180 分鐘補休額度
```

這時**不要直接算 NT$多少**。

---

### 到薪資結算時

例如 8 月薪資結算：

```text
8月出勤
    ↓
正常工時
    ↓
加班
    ↓
請假
    ↓
補休
    ↓
薪資規則
    ↓
8月薪資
```

這時才計算：

```text
基本薪資
+ 加班費
+ 補休到期轉薪資
+ 津貼
- 請假扣款
- 其他扣款
...
= 實發薪資
```

---

## 為什麼我強烈建議事後算？

因為薪資不是單一事件決定的。

例如員工 8 月：

```text
基本薪資：40,000

加班：
8/5 2小時
8/10 3小時
8/20 4小時

請假：
8/15 4小時

補休：
8/10 加班3小時 → 轉補休
```

最後薪資要一起結算。

而且還可能遇到：

- 月中調薪
- 到職
- 離職
- 勞健保
- 所得稅
- 加班倍率
- 津貼
- 扣款
- 補休到期
- 公司不同薪資週期

所以**事件當下直接算錢會非常難維護**。

---

# 但有一個很重要的區別

我會把系統分成：

### ① 即時產生「權利／額度」

例如：

```text
加班 3 小時
↓
立即產生
補休 3 小時
```

這個可以當下產生。

---

### ② 薪資金額

例如：

```text
3小時補休
↓
到底值多少錢？
```

**不要當下算。**

等薪資結算時才算。

---

# ③ 薪資結算後要「鎖定」

例如：

```text
8月份薪資
↓
計算
↓
審核
↓
結算
↓
鎖定
```

鎖定後就不能因為有人偷偷改了 8/5 的班表，就讓 8 月薪資跟著變。

如果真的需要修改：

```text
已結算薪資
    ↓
薪資更正
    ↓
補發 / 扣回
```

而不是重新把歷史薪資整包重算。

---

## 所以我建議我們的核心模型是

```text
「事件即時記錄」
        ↓
「期間結算」
        ↓
「薪資鎖定」
```

也就是：

```text
打卡
請假
加班
補休
調班
        ↓
產生 HR 事件
        ↓
薪資結算期
        ↓
Payroll Calculation
        ↓
Payroll Result
        ↓
鎖定
```

### 對我們現在的 Schema 影響

所以前面我才會建議：

```text
compensatory_leave_credits
```

**存「補休分鐘數」**

而不是：

```text
補休價值 NT$xxx
```

因為：

> **分鐘數是事件當下就確定的；金額是薪資結算時才確定的。**

這個原則我認為應該直接定為這套 HR SaaS 的核心設計原則之一。

---

## 2026/08/22 22:07:36

### 使用者

可是如果調薪了，就不應該把當時核發的補休時間，變成後期條新的薪水來補

### 助理

對，你這個判斷是對的。**這也是為什麼補休不能只記「180 分鐘」而已。**

你的意思是：

> 8 月加班產生補休，當時員工薪資是 40,000。  
> 9 月調薪到 45,000。  
> 8 月產生的補休到了 10 月才到期，轉薪資時，**不能用 45,000 的薪資標準重新計算**。

這個需求我同意，而且會讓我**修改前面「薪資事後才計算」的設計**。

---

# 正確做法：補休產生時，就要「鎖定計價基準」

但不是直接把最終金額算死。

例如：

```text id="h4t0as"
8/10

加班 3 小時
↓
轉補休
↓
補休 180 分鐘

當時：
基本薪資 = 40,000
計價基準 = 40,000
```

系統要保存：

```text id="b3j7r4"
source_overtime_id
earned_minutes = 180

pay_rate_snapshot
...
```

這叫做 **Snapshot（快照）**。

---

# 為什麼要快照？

假設：

```text id="2b9f9d"
8月
薪資 40,000
↓
產生 3 小時補休

9月
薪資調整為 45,000

10月
補休到期
```

正確應該：

```text id="h1s0sl"
補休來源：
8月

計價基準：
8月薪資制度 / 薪資基準

而不是：
10月目前薪資
```

所以：

```text id="n5f5m0"
補休
│
├─ 產生時間
├─ 產生分鐘
├─ 來源加班
└─ 產生當下的薪資計價快照
```

---

# 但是「薪資快照」不要只存一個薪資

這裡要特別小心。

不能只做：

```text id="h8z9pw"
pay_rate = 40000
```

因為未來可能有：

```text id="2j0qz7"
基本薪資
職務加給
固定津貼
時薪
加班倍率
...
```

所以比較合理的是：

## `compensatory_leave_credits`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 補休額度 |
| `employee_id` | `uuid` | 員工 |
| `source_overtime_id` | `uuid` | 來源加班 |
| `earned_minutes` | `integer` | 取得分鐘 |
| `used_minutes` | `integer` | 已使用分鐘 |
| `settled_minutes` | `integer` | 已轉薪資分鐘 |
| `remaining_minutes` | `integer` | 剩餘分鐘 |
| `earned_at` | `datetime` | 取得時間 |
| `expire_at` | `datetime` | 到期時間 |
| `status_code` | `integer` | 狀態 |
| `created_at` | `datetime` | 建立 |
| `updated_at` | `datetime` | 更新 |

然後另外建立：

## `compensatory_leave_rate_snapshots`

這張專門保存：

> **這筆補休產生當下，使用什麼薪資基準。**

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | ID |
| `compensatory_leave_credit_id` | `uuid` | 補休額度 |
| `calculation_type_code` | `integer` | 計算類型 |
| `base_amount` | `decimal` | 計算基準 |
| `rate` | `decimal` | 計算倍率 |
| `effective_from` | `datetime` | 基準生效時間 |
| `effective_to` | `datetime` | 基準失效時間 |
| `snapshot_data` | `json` | 計價所需完整快照 |
| `created_at` | `datetime` | 建立時間 |

---

# 這樣就能做到「薪資事後算，但基準不變」

這個概念很重要。

不是：

> **當下完全不算**

而是：

> **當下確定計算所需的基準，事後才產生最終薪資結果。**

例如：

```text id="qgl5kf"
8/10
加班 3 小時
↓
產生 180 分鐘補休
↓
鎖定當時薪資計價基準
```

9 月：

```text id="7lwh9k"
調薪
40,000 → 45,000
```

10 月：

```text id="g4x4hc"
補休到期
↓
讀取 8/10 的 rate snapshot
↓
計算
↓
進入薪資
```

而不是讀現在的 45,000。

---

# 這個概念其實不只適用補休

我會建議我們直接把它變成**薪資系統的核心原則**：

### 「產生事件時鎖定計算基準，薪資結算時產生金額。」

例如：

| 事件 | 當下鎖定 | 薪資結算時 |
|---|---|---|
| 加班 | 加班計算基準 | 算加班費 |
| 補休 | 補休計價基準 | 到期轉薪資 |
| 津貼 | 適用規則 | 算金額 |
| 扣款 | 扣款規則 | 算金額 |

這樣即使：

```text id="e8l0h7"
8月 40,000
9月 45,000
10月 50,000
```

8 月產生的補休，**永遠不會因為後面的調薪而偷偷變貴。**

---

### 所以我會修正我們前面的設計

**補休現在正式增加「產生時薪資計價基準快照」。**

而且我認為這個設計非常重要，甚至比「到期轉薪資」本身還重要。

否則未來一遇到**調薪、職務異動、薪資結構變更**，歷史補休的金額就會亂掉。

---

## 2026/08/22 22:08:57

### 使用者

基準值怎麼定義，如果套入UI，是在核發當下去勾選基準值?

### 助理

對，但我會再修正一下：**不要讓使用者直接在「核發補休」畫面手動選一個金額當基準值。**

比較好的做法是：

> **UI 讓管理者選擇「補休計價方式」，系統自動帶出當下適用的薪資基準，使用者可以確認，但不是自由輸入。**

這樣比較不容易出錯。

---

## 例如核發補休的畫面

員工：

> 王小明

加班：

> 2026/08/10 18:00～21:00  
> 加班 180 分鐘

### 「加班處理方式」

```text
○ 發放加班費
● 轉補休
```

選擇「轉補休」後，下面出現：

### 補休計價基準

```text
計價方式：
[ 依加班發生時薪資基準 ▼ ]

計算基準：
基本薪資：40,000 元
適用日期：2026/08/01 ～ 2026/08/31

時薪基準：
40,000 ÷ 240 = 166.67 元
```

這些是**系統自動抓出來的**。

管理者可以看到，但不應該直接改成：

```text
45,000
```

---

# 那「基準值」到底是什麼？

這要先定義清楚。

我不建議我們直接把：

> 「基本薪資」

當成補休的基準。

因為未來薪資結構可能是：

```text
基本薪資
職務加給
固定津貼
...
```

而不同公司可能有不同計算方式。

所以我會把它拆成：

### ① 計價方式

例如：

```text
1 = 依加班發生時薪資基準
2 = 依固定時薪
3 = 依薪資制度規則
```

這是**規則**。

---

### ② 實際基準

系統依照上面的規則，找到：

```text
2026/08/10
這個員工
當時適用的薪資資料
```

例如：

```text
基本薪資       40,000
固定職務加給    5,000
```

再依薪資制度計算。

---

# 所以 UI 最好是這樣

### Step 1：主管核准加班

```text
加班時間
2026/08/10 18:00 ～ 21:00

實際加班
180 分鐘
```

### Step 2：選擇處理方式

```text
加班處理

○ 加班費
● 補休
```

### Step 3：系統自動帶出

```text
補休時數
3 小時

計價基準
[依加班發生時薪資基準]

系統判定：
基本薪資       40,000
固定職務加給     5,000
適用薪資制度     A制度
```

然後：

> **「確認核發補休」**

---

# 但我會多加一個「覆核」概念

因為實務上可能有特殊情況。

例如 HR 發現：

> 這個人的薪資資料還沒建好。

那系統不能默默抓一個錯的數字。

應該顯示：

```text
⚠ 無法取得 2026/08/10
適用的薪資計價基準

請先確認薪資資料
```

而不是讓 HR 自己隨便輸入。

---

# 還有一個更重要的地方

你前面說：

> **薪資是事後算。**

我會保留這個原則。

所以「核發補休」當下：

### 不算最終薪資金額

而是鎖：

```text
計價方式
+
適用薪資制度
+
適用薪資資料
+
必要的計算參數
```

也就是：

```text
              核發補休
                  │
                  ▼
        ┌─────────────────┐
        │ 計價基準 Snapshot │
        ├─────────────────┤
        │ 計價方式          │
        │ 薪資制度          │
        │ 基本薪資          │
        │ 固定薪資項目       │
        │ 計算參數          │
        └─────────────────┘
                  │
                  ▼
             補休額度
                  │
             到期未使用
                  │
                  ▼
             薪資結算
                  │
                  ▼
        使用當時已鎖定的基準
```

---

## 我甚至建議 UI 不要叫「基準值」

對 HR 使用者來說，「基準值」太技術。

UI 可以叫：

> **補休計價依據**

例如：

```text
補休計價依據
● 依加班發生時適用薪資
○ 依公司補休計價規則
```

然後下面：

```text
系統自動判定

適用薪資基準
2026/08/10

基本薪資：40,000 元
固定薪資項目：5,000 元
```

---

### 最後有一個我認為我們現在必須先定的問題

**「補休到期轉薪資」到底是按照「加班當時的薪資基準」，還是按照「補休產生當下已核定的補休價值」？**

這兩個在某些薪資制度下可能不完全一樣。

如果我們要把 Schema 做到可以真正落地，我建議下一步就把**「補休計價基準的規則」定死，再來設計 UI 和 Schema**。

---

## 2026/08/22 22:09:24

### 使用者

在幫我思考是否還有問題

### 助理

有，而且我重新把「加班 → 補休 → 調薪 → 到期轉薪資」整條流程想過一次後，**有幾個問題現在就要定義**。不然做到薪資時，很容易又回頭改 Schema。

我認為最重要的是下面 8 個。

---

## 1. 「補休的計價基準」到底鎖什麼？

這是目前最大的問題。

不能只鎖：

```text
基本薪資 = 40,000
```

因為可能還有：

- 職務加給
- 固定津貼
- 其他會影響加班計算的項目
- 時薪換算方式
- 加班倍率

所以我建議：

> **補休核發時，不只保存金額，而是保存「當時適用的計價規則 + 計算所需資料 Snapshot」。**

例如：

```text
補休 180 分鐘

計價規則：
依加班發生時薪資基準

薪資基準版本：
2026-08-01 version 3

基本薪資：
40,000

時薪換算：
166.67

其他計算參數：
......
```

未來薪資改變，不影響這筆補休。

---

# 2. 加班「轉補休」之後，還能不能改成「加班費」？

這個一定會發生。

例如員工：

> 8/10 加班 4 小時 → 原本選補休

後來：

> 員工說不要補休了，改領錢。

那能不能改？

我建議：

### 在補休尚未使用、尚未結算之前

**可以申請轉換。**

流程：

```text
補休
↓
申請轉加班費
↓
主管核准
↓
撤銷補休額度
↓
轉入薪資
```

但必須留下完整交易紀錄。

---

# 3. 補休已經用掉，還能改嗎？

例如：

```text
補休 8 小時
↓
已使用 4 小時
↓
剩 4 小時
```

這時不能把整筆補休轉成薪資。

只能：

```text
已使用 4 小時
→ 保持補休

剩餘 4 小時
→ 才能依規則處理
```

所以補休一定要支援**部分使用**。

這也是為什麼我前面建議：

```text
earned_minutes
used_minutes
settled_minutes
```

---

# 4. 加班本身能不能撤銷？

也要處理。

例如：

```text
主管核准加班 4 小時
↓
轉補休 4 小時
↓
後來發現其實只加班 2 小時
```

那就不能只改：

```text
earned_minutes = 120
```

應該留下：

```text
原始：
+240

更正：
-120

最後：
120
```

所以：

> **加班、補休、薪資全部都要採「異動紀錄」，不要直接覆蓋歷史。**

---

# 5. 補休到期日怎麼決定？

這個目前還沒有真正定義。

例如：

```text
加班日：2026/08/10
補休產生：2026/08/15
```

到底：

```text
A. 從加班日開始算？
B. 從補休核發日開始算？
C. 公司設定固定期限？
D. 依法律規則？
```

我建議 Schema **不要寫死計算方式**。

保存：

```text
earned_at
expire_at
```

就好。

真正怎麼算 `expire_at`，由補休規則決定。

---

# 6. 補休到期「轉薪資」時，到底進哪一期薪資？

這個非常重要。

例如：

```text
補休到期日
2026/08/31
```

但薪資週期可能是：

```text
8/1～8/31
9/1～9/30
```

也可能公司是：

```text
每月 21 日～20 日
```

所以不能簡單：

```text
expire_at → 當月薪資
```

應該是：

```text
補休到期
↓
產生「待薪資結算事件」
↓
由 Payroll 找到適用的薪資週期
```

這樣才乾淨。

---

# 7. 如果薪資已經結算，補休才被發現過期怎麼辦？

例如：

```text
8月薪資
↓
已結算
↓
9月發現有一筆補休 4 小時忘了轉薪
```

不能回頭修改已鎖定的 8 月薪資。

應該：

```text
8月補休到期
↓
產生待結算項目
↓
9月薪資
↓
補發
```

所以我們之後的 Payroll 必須支援：

> **補發 / 更正薪資來源。**

這個現在不一定要設計完整，但 Schema 要留得住。

---

# 8. 最後一個：公司規則可能不同

這是我認為最容易被低估的問題。

不同公司可能：

```text
公司 A：
加班 1 小時 → 補休 1 小時

公司 B：
加班依倍率產生補休

公司 C：
某些加班可以補休
某些加班只能領錢

公司 D：
補休期限不同
```

所以：

> **不要把「1 小時加班 = 1 小時補休」寫死在程式裡。**

應該有：

```text
overtime_rules
```

或者至少讓加班規則能定義：

```text
overtime_compensation_type
compensatory_leave_rate
expiry_rule
```

---

# 我重新整理後，整個設計應該變成

```text
                 加班
                  │
                  ▼
          overtime_request
                  │
          ┌───────┴────────┐
          ▼                ▼
       加班費             補休
                           │
                           ▼
             compensatory_leave_credit
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
             使用補休              到期
                │                     │
                ▼                     ▼
          leave_request          待薪資結算
                                      │
                                      ▼
                                   Payroll
```

而每個關鍵節點都留下：

```text
來源
時間
規則版本
計價基準
異動紀錄
```

---

## 我現在最建議你先定的，其實只有 3 件事

不要一次把所有薪資規則都做完。

### A. 補休產生時

**鎖定「計價基準 Snapshot」**

### B. 補休到期時

**剩餘時間一定轉成薪資，不得消失**

### C. 薪資調整後

**歷史補休不重新套用新薪資**

這三條一旦確定，後面的 Schema 就會穩很多。

而且我會建議我們下一步**不要急著再加 Schema**，先把這三條規則加上「加班轉補休 → 補休使用 → 調薪 → 到期轉薪資」跑一個完整案例，看看有沒有漏洞。這樣比一直堆欄位更容易發現設計問題。

