# 2026-08-22 對話逐字稿 — Part 006

---

## 2026/08/22 21:37:39

### 使用者

人事成本的部分

### 助理

可以，既然薪資本身已經先定下來，現在可以開始討論**人事成本**。

我建議先把概念切乾淨：

> **薪資 = 員工拿到多少 / 被扣多少**
>
> **人事成本 = 公司為這個員工實際承擔多少成本**

所以人事成本不要塞進 `payrolls` 裡面。

---

# 一、人事成本會有哪些？

先不要急著做 Schema，我們先列需求。

### ① 員工薪資

例如：

```text
基本薪資       50,000
主管加給        5,000
交通津貼        2,000
```

這些通常也是公司成本。

---

### ② 公司負擔的勞健保

例如：

```text
公司負擔勞保
公司負擔健保
```

注意：

```text
員工負擔勞保
員工負擔健保
```

是**員工薪資扣款**，不是另外的人事成本。

但：

```text
公司負擔勞保
公司負擔健保
```

是公司額外成本。

---

### ③ 勞退

例如：

```text
公司提撥勞工退休金
```

這也是公司人事成本。

而且它跟員工實際拿到的薪資不同。

---

### ④ 公司福利

例如：

```text
員工團保
生日禮金
員工餐費
交通補助
福利金
健康檢查
```

這些有些可能透過薪資發放，有些根本不會進薪資單。

所以不能只從 Payroll 推導。

---

### ⑤ 獎金

例如：

```text
年終獎金
三節獎金
績效獎金
專案獎金
```

這些通常也是公司的人事成本。

---

### ⑥ 其他員工相關成本

未來可能有：

```text
教育訓練
證照費
招募成本
制服
設備
員工福利
```

但這些我建議**不要現在全部塞進人事成本**。

因為已經開始往「管理會計」方向走。

---

# 二、我建議人事成本先分成 4 大類

可以先定義概念：

```text
1 = 薪資成本
2 = 公司法定負擔
3 = 獎金福利
4 = 其他人事成本
```

例如：

```text
薪資成本
├── 基本薪資
├── 職務加給
├── 主管加給
└── 津貼

公司法定負擔
├── 公司負擔勞保
├── 公司負擔健保
└── 勞退

獎金福利
├── 年終
├── 績效獎金
├── 團保
└── 員工福利

其他人事成本
├── 教育訓練
└── 其他
```

---

# 三、最重要的是「人事成本要能追到員工」

例如：

```text
2026/08 王小明

薪資成本          57,000
公司勞保           2,000
公司健保           3,000
勞退               3,000
------------------------
人事成本          65,000
```

這樣公司才可以查：

### 員工成本

```text
王小明
每月人事成本 65,000
```

也可以進一步查：

### 部門成本

```text
研發部

王小明    65,000
李小華    58,000
陳大明    72,000
----------------
部門成本 195,000
```

這也是為什麼我們之前決定：

> **員工同一時間只能有一個部門，但部門需要保留歷史。**

這個設計到了人事成本會非常有用。

---

# 四、還有一個很重要的問題：成本月份

例如：

```text
8 月薪資
```

但可能：

```text
8/31 發薪
```

公司負擔的勞健保可能又是另一個計算期間。

所以人事成本一定要有：

```text
cost_date
```

或：

```text
cost_period
```

而不能單純看：

```text payroll.pay_date
```

否則未來做月報表可能會出問題。

---

# 五、我目前會建議先做這兩張

## `personnel_cost_items`

### 說明

**人事成本項目定義。**

用來定義：

```text
薪資
公司勞保
公司健保
勞退
年終獎金
團保
教育訓練
其他
```

---

## `personnel_costs`

### 說明

**實際人事成本紀錄。**

例如：

```text
2026/08
王小明

基本薪資       50,000
主管加給        5,000
交通津貼        2,000
公司勞保        2,000
公司健保        3,000
勞退            3,000
```

最後：

```text
人事成本 = 65,000
```

---

# 六、但這裡我建議先停一下

因為人事成本有一個關鍵決策：

### A. 人事成本「從 Payroll 自動產生」

```text
Payroll
   ↓
人事成本
```

例如薪資 57,000，自動產生薪資成本。

### B. 人事成本可以「獨立建立」

例如：

```text
教育訓練費 5,000
團保 1,000
```

它不一定有 Payroll。

### C. 兩種都支援

我**最推薦 C**：

```text
Payroll
   ↓
自動產生人事成本

另外也可以
手動建立人事成本
```

這樣 SaaS 才不會被限制死。

而且之後可以做到：

```text
公司
 ├── 員工
 │    └── 人事成本
 │
 ├── 部門
 │    └── 人事成本
 │
 └── 月份
      └── 人事成本報表
```

如果你同意這個方向，下一步我們就可以正式把 **`personnel_cost_items` + `personnel_costs` Schema** 定下來。

---

## 2026/08/22 21:38:41

### 使用者

可以

### 助理

好，那我們就正式定義人事成本的 Schema。

我建議先維持**簡單、可擴充**，不要一開始把會計系統的東西全部塞進來。

目前確定兩張核心表：

```text id="f6qgn6"
personnel_cost_items
        │
        ▼
personnel_costs
```

另外可以由 Payroll 自動產生，也可以手動建立。

---

# 1. `personnel_cost_items`

### 說明

**人事成本項目定義表。**

用來定義公司有哪些「人事成本類型」。

例如：

```text id="7nv5r8"
薪資
公司負擔勞保
公司負擔健保
勞退
年終獎金
績效獎金
員工團保
教育訓練
其他人事成本
```

系統可以提供預設項目，公司也可以建立自己的項目。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 人事成本項目 ID |
| `company_id` | `uuid` | 所屬公司 ID，NULL 表示系統預設 |
| `code` | `string` | 成本項目代碼 |
| `name` | `string` | 成本項目名稱 |
| `type_code` | `integer` | 成本類型代碼 |
| `is_active` | `boolean` | 是否啟用 |
| `description` | `string` | 項目說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

## 預設項目

可以先提供：

```text id="e14ms0"
1  SALARY              薪資
2  LABOR_INSURANCE     公司負擔勞保
3  HEALTH_INSURANCE    公司負擔健保
4  LABOR_PENSION       勞退
5  BONUS               獎金
6  EMPLOYEE_INSURANCE  員工團保
7  TRAINING            教育訓練
8  BENEFIT             員工福利
9  OTHER               其他人事成本
```

這些只是**系統預設值**，公司仍可以建立：

```text id="b9a3rq"
交通補助
住宿補助
證照補助
員工活動
```

---

# 2. `personnel_costs`

### 說明

**實際人事成本紀錄。**

一筆代表：

> 某公司在某個成本期間，為某位員工產生的一筆人事成本。

例如：

```text id="bqz6d3"
2026/08

王小明

薪資            57,000
公司勞保         2,000
公司健保         3,000
勞退             3,000
----------------------
人事成本        65,000
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 人事成本 ID |
| `company_id` | `uuid` | 公司 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `personnel_cost_item_id` | `uuid` | 人事成本項目 ID |
| `payroll_id` | `uuid` | 關聯薪資單，可 NULL |
| `cost_date` | `date` | 成本日期 |
| `cost_period` | `string` | 成本期間，例如 `202608` |
| `amount` | `decimal` | 成本金額 |
| `source_type_code` | `integer` | 成本來源 |
| `description` | `string` | 成本說明 |
| `created_by` | `uuid` | 建立人員 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 3. `source_type_code`

我們不使用 ENUM。

先定義：

```text id="e20g3g"
1 = Payroll 自動產生
2 = 手動建立
```

例如：

### Payroll 自動產生

```text id="0t0d8q"
薪資
57,000

source_type_code = 1
payroll_id = XXXXX
```

### 手動建立

例如公司幫員工支付教育訓練：

```text id="m1p68w"
教育訓練
5,000

source_type_code = 2
payroll_id = NULL
```

---

# 4. 為什麼 `payroll_id` 可以 NULL？

因為不是所有人事成本都會經過薪資。

例如：

```text id="lhb8k2"
員工教育訓練費
5,000
```

不會出現在：

```text
payroll
```

但是它仍然是公司的人事成本。

所以：

```text id="c5gn5z"
payroll_id = NULL
```

是合理的。

---

# 5. 這樣就能做很重要的報表

例如：

### 員工月成本

```text id="3z5v4j"
王小明
2026/08

薪資            57,000
公司勞保         2,000
公司健保         3,000
勞退             3,000
教育訓練         5,000
----------------------
總人事成本       70,000
```

---

### 部門月成本

因為我們已經有人員的部門歷史：

```text id="wxq9t8"
研發部

王小明          70,000
李小華          62,000
陳大明          75,000
----------------------
研發部人事成本 207,000
```

這裡**不需要在 `personnel_costs` 再存 `department_id`**。

因為員工的部門本身有歷史，可以依照 `cost_date` 找出當時所屬部門。

這樣比較不會產生：

```text id="0gl5d2"
員工現在在研發部
但舊的人事成本被算到業務部
```

---

# 6. 最終人事成本架構

```text id="i9xw6x"
                 personnel_cost_items
                         │
                         ▼
                  personnel_costs
                    │          │
                    │          └── employee
                    │
                    └────────── payroll
```

Payroll：

```text id="xv8f0y"
salary
   ↓
payroll
   ↓
自動產生
   ↓
personnel_costs
```

其他成本：

```text id="w3b0a6"
教育訓練
員工福利
團保
其他
   ↓
手動建立
   ↓
personnel_costs
```

---

## 目前我建議的人事成本邊界

**這次納入：**

- 員工薪資成本
- 公司負擔勞保
- 公司負擔健保
- 勞退
- 獎金
- 員工福利
- 團保
- 教育訓練
- 其他人事成本
- 員工成本統計
- 部門成本統計
- 月份成本統計
- Payroll 自動產生成本
- 手動新增成本

**這次先不做：**

- 會計分錄
- 科目
- 成本中心
- 部門預算
- 專案成本會計
- 損益
- 財務報表

這樣它還是**HR 的人事成本功能**，而不是開始變成一套會計系統。

---

## 2026/08/22 21:39:39

### 使用者

記錄起來

### 助理

已記錄。

目前 HR SaaS 的**薪資 + 人事成本**設計會以目前確認的 Schema 為基準，之後可以直接往下一個模組延伸。

---

## 2026/08/22 21:42:32

### 使用者

再來討論打卡，員工打卡、可以自己撤銷打卡，上班打卡後才能打下班卡，兩個打卡都可以撤銷，因為可能誤打，忘記打卡則可以申請補登記，分開登記，因為有能可能只打下卡，忘了打上班卡，打卡藥可以記錄GPS這不強制

### 助理

可以，這一塊我建議先把「**正常打卡**」和「**補登 / 撤銷**」分開，不要把所有東西塞進一張打卡表。

你目前提出的規則我先整理：

### 已確定需求

1. 員工可以打「上班卡」
2. **有上班卡後才能打下班卡**
3. 上班卡可以自行撤銷
4. 下班卡可以自行撤銷
5. 撤銷是因為可能誤打
6. 忘記打卡可以申請補登
7. **上班、下班補登要分開申請**
   - 忘記上班卡 → 只補上班卡
   - 忘記下班卡 → 只補下班卡
   - 兩個都忘記 → 分別補兩筆
8. 打卡可以記錄 GPS
9. GPS **不是強制**
10. 打卡紀錄需要能追蹤歷史，不應該直接刪除

---

# 一、我建議先有 3 張核心 Schema

```text
attendance_records
        │
        ├── 正常打卡
        │
        └── 撤銷

attendance_correction_requests
        │
        └── 補登申請

attendance_settings
        │
        └── 公司打卡規則
```

---

# 1. `attendance_records`

### 說明

**員工實際打卡紀錄。**

不管是正常打卡還是補登，最後都應該形成正式的打卡紀錄。

例如：

```text
王小明

2026/08/22 08:58
上班

2026/08/22 18:03
下班
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 打卡紀錄 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `work_date` | `date` | 工作日期 |
| `attendance_type_code` | `integer` | 打卡類型：1=上班、2=下班 |
| `clocked_at` | `datetime` | 實際打卡時間 |
| `latitude` | `decimal` | GPS 緯度，可 NULL |
| `longitude` | `decimal` | GPS 經度，可 NULL |
| `location_accuracy` | `decimal` | GPS 精確度，可 NULL |
| `source_type_code` | `integer` | 打卡來源 |
| `status_code` | `integer` | 打卡狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

### `source_type_code`

先可以定：

```text
1 = 員工正常打卡
2 = 補登
3 = 管理者補登
```

### `status_code`

例如：

```text
1 = 有效
2 = 已撤銷
```

**不直接 DELETE 打卡紀錄。**

---

# 二、為什麼要有 `work_date`？

這個很重要。

因為：

```text
clocked_at = 2026/08/22 08:58
```

不一定代表「今天的工作」。

未來如果支援：

```text
跨日班
夜班
22:00 → 隔日 06:00
```

就會需要區分：

```text
work_date
```

例如：

```text
work_date = 2026/08/22

上班：2026/08/22 22:00
下班：2026/08/23 06:00
```

所以我建議現在就留。

---

# 三、`attendance_correction_requests`

### 說明

**補登打卡申請。**

這張不是打卡紀錄，而是：

> 員工說「我忘記打卡，請幫我補登」。

例如：

```text
王小明

2026/08/22
忘記上班打卡

申請補登：
08:57
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 補登申請 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `work_date` | `date` | 工作日期 |
| `attendance_type_code` | `integer` | 補登類型：1=上班、2=下班 |
| `requested_clocked_at` | `datetime` | 申請補登時間 |
| `reason` | `string` | 補登原因 |
| `status_code` | `integer` | 申請狀態 |
| `reviewed_by` | `uuid` | 審核人員 |
| `reviewed_at` | `datetime` | 審核時間 |
| `review_comment` | `string` | 審核說明 |
| `attendance_record_id` | `uuid` | 審核通過後產生的打卡紀錄 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 四、為什麼補登要獨立？

因為：

```text
正常打卡
```

和：

```text
「我忘記打卡，現在申請補登」
```

是完全不同的事情。

例如：

```text
08:55 正常打上班卡
18:05 忘記打下班卡
```

員工提出：

```text
補登下班
18:03
```

審核通過後才產生：

```text
attendance_records

18:03
下班
source_type_code = 2
```

這樣歷史很乾淨。

---

# 五、撤銷怎麼處理？

這裡我建議**不要另外建立撤銷表**。

直接在 `attendance_records`：

```text
status_code = 2
```

並另外記錄：

```text
cancelled_at
cancelled_by
cancel_reason
```

所以我會把 Schema 稍微調整成：

| 欄位 | 型態 | 說明 |
|---|---|---|
| `status_code` | `integer` | 1=有效、2=已撤銷 |
| `cancelled_at` | `datetime` | 撤銷時間，可 NULL |
| `cancelled_by` | `uuid` | 撤銷人員，可 NULL |
| `cancel_reason` | `string` | 撤銷原因，可 NULL |

---

# 六、你提出的「上班後才能下班」很重要

系統規則：

```text
沒有上班卡
       ↓
不能打下班卡
```

例如：

```text
08:55 上班 ✅
18:03 下班 ✅
```

但是：

```text
18:03 下班 ❌
```

因為沒有有效的上班紀錄。

---

## 但撤銷後要怎麼辦？

這裡有一個很重要的規則需要定。

例如：

```text
08:55 上班
18:03 下班
```

員工發現：

> 18:03 是誤打。

撤銷：

```text
18:03 下班 → 已撤銷
```

這時候：

```text
有效上班卡：08:55
有效下班卡：無
```

所以他還是可以重新打：

```text
18:10 下班
```

這是合理的。

---

# 七、但是「上班卡撤銷」會比較麻煩

例如：

```text
08:55 上班
18:03 下班
```

員工把：

```text
08:55 上班
```

撤銷。

這時候會變成：

```text
有效上班卡：無
有效下班卡：18:03
```

這是一個**不完整狀態**。

所以我建議系統：

> **允許撤銷上班卡，但撤銷後如果還存在有效下班卡，必須提示異常。**

而不是偷偷把下班卡一起刪掉。

因為使用者可能只是誤撤銷上班卡。

---

# 八、GPS

你說 GPS：

> 可以記錄，不強制。

我同意。

所以：

```text
latitude
longitude
location_accuracy
```

全部：

```text
NULLABLE
```

例如：

```text
正常打卡

08:58
GPS：
25.0478
121.5318
精度：15m
```

也可以：

```text
08:58
GPS：NULL
```

仍然是有效打卡。

**不要把「沒有 GPS」直接判定成異常。**

---

# 九、我會再加一個「打卡來源」

因為 SaaS 未來很可能不只有：

```text
手機
```

可能有：

```text
Web
手機 App
平板
打卡機
管理者
```

所以：

```text
source_type_code
```

我會建議之後定成例如：

```text
1 = Web
2 = Mobile
3 = 打卡機
4 = 管理者
```

補登則透過另外的 `attendance_correction_requests` 判斷來源。

---

# 十、目前打卡 Schema

```text
attendance_settings
        │
        │ 公司打卡規則
        ▼
attendance_records
        │
        ├── 上班
        ├── 下班
        └── 撤銷

attendance_correction_requests
        │
        │ 審核通過
        ▼
attendance_records
```

這樣的好處是：

**正常打卡**

```text
attendance_records
```

**忘記打卡**

```text
attendance_correction_requests
        ↓
審核
        ↓
attendance_records
```

**誤打**

```text
attendance_records
        ↓
status = 已撤銷
```

整個歷史就都保留下來。

---

我目前認為這套打卡 Schema **還有一個一定要討論的核心：班別 / 出勤規則**。

因為如果沒有班別，我們只能知道：

> 「08:58 打卡、18:03 打卡」

但不知道：

> **這個人今天應該幾點上班、幾點下班、遲到幾分鐘、早退幾分鐘。**

而這又會直接影響後面的**加班與薪資計算**。

---

## 2026/08/22 21:44:55

### 使用者

所以在設計班表

### 助理

對，**班表應該獨立設計**，而且我會建議不要把「班表」直接塞進打卡資料。

目前可以把關係理解成：

```text
班別設定
   ↓
班表
   ↓
員工當天應出勤規則
   ↓
員工打卡
   ↓
出勤結果
   ↓
薪資計算
```

但這裡有一個很重要的區分：

### 1. 班別 `shifts`

定義「這是一個什麼班」。

例如：

```text
一般班
上班 08:00
下班 17:00

早班
上班 07:00
下班 16:00

晚班
上班 14:00
下班 23:00

夜班
上班 22:00
下班 06:00
```

這是**規則定義**。

---

### 2. 班表 `employee_schedules`

定義：

> **哪個員工哪一天要上哪個班。**

例如：

```text
王小明

2026/08/24 → 一般班
2026/08/25 → 一般班
2026/08/26 → 休息
2026/08/27 → 一般班
```

這才是真正的「班表」。

---

### 3. 打卡 `attendance_records`

才是實際發生的事情：

```text
應上班 08:00
實際上班 08:07

應下班 17:00
實際下班 17:05
```

因此系統才能判斷：

```text
遲到 7 分鐘
早退 0 分鐘
```

---

## 我建議班表至少拆成這兩層

```text
shifts
    ↓
employee_schedules
```

而不是：

```text
employee_schedules
裡面直接寫 08:00、17:00
```

因為未來可能有：

```text
一般班
早班
晚班
夜班
彈性班
```

而且同一個班別可以給很多員工使用。

---

## 還有一個很重要的問題

我們之後要決定：

**班表是「每天指定」，還是支援「週期性班表」。**

例如一般公司可能是：

```text
週一～週五 → 一般班
週六 → 休息
週日 → 休息
```

但工廠可能：

```text
早班 → 4天
休 → 2天
晚班 → 4天
休 → 2天
```

甚至：

```text
A班
B班
C班
```

輪班。

所以我會建議下一步先**只討論班別 Schema 和班表 Schema**，先不要碰打卡計算邏輯。

而且這部分會直接影響之後的：

- 遲到
- 早退
- 加班
- 跨日班
- 休息日
- 國定假日
- 請假
- 薪資計算

所以班表這層確實值得先設計好。

---

## 2026/08/22 21:45:39

### 使用者

最大化設計

### 助理

可以。如果我們現在要用**「最大化設計」**來做，我會把班表設計成可以支援一般公司、輪班、跨日、彈性工時、休息時間、加班判定，而且不要為了簡單而把未來需求鎖死。

但我會維持一個原則：

> **班別是規則，班表是安排，實際出勤是結果。**

這三件事情不能混在一起。

---

# 一、整體架構

我建議班表相關先設計成：

```text
shift_definitions
        │
        ├── shift_work_periods
        │
        └── shift_breaks
                │
                ▼
        employee_schedules
                │
                ▼
        attendance_records
                │
                ▼
        attendance_results
```

另外：

```text
schedule_rules
schedule_rule_details
```

處理「週一到週五」、「四休二」、「輪班」等週期性規則。

以及：

```text
holiday_calendars
holiday_calendar_dates
```

處理國定假日、公司休假日。

---

# 1. `shift_definitions`

## 說明

**班別主檔。**

定義公司有哪些班別。

例如：

```text
一般班
早班
晚班
夜班
彈性班
```

注意：

> 這張表只定義「班別是什麼」，不負責決定哪一天誰上班。

## Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 班別 ID |
| `company_id` | `uuid` | 所屬公司 ID |
| `code` | `string` | 班別代碼 |
| `name` | `string` | 班別名稱 |
| `work_type_code` | `integer` | 工時類型 |
| `is_overnight` | `boolean` | 是否跨日 |
| `is_flexible` | `boolean` | 是否為彈性工時 |
| `required_work_minutes` | `integer` | 應工作分鐘數 |
| `description` | `string` | 班別說明 |
| `is_active` | `boolean` | 是否啟用 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 2. `shift_work_periods`

這張我認為**非常重要**。

因為一個班不一定只有：

```text
08:00 → 17:00
```

可能是：

```text
08:00 → 12:00
13:00 → 17:00
```

甚至：

```text
22:00 → 02:00
03:00 → 06:00
```

所以不要在 `shift_definitions` 直接放上班時間。

## 說明

**班別工作時段。**

一個班別可以有多個工作時段。

## Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 工作時段 ID |
| `shift_id` | `uuid` | 班別 ID |
| `sequence` | `integer` | 時段順序 |
| `start_time` | `time` | 開始時間 |
| `end_time` | `time` | 結束時間 |
| `start_day_offset` | `integer` | 開始日偏移 |
| `end_day_offset` | `integer` | 結束日偏移 |
| `required_minutes` | `integer` | 此時段應工作分鐘 |
| `late_grace_minutes` | `integer` | 遲到寬限分鐘 |
| `early_leave_grace_minutes` | `integer` | 早退寬限分鐘 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

例如夜班：

```text
start_time = 22:00
end_time = 06:00

start_day_offset = 0
end_day_offset = 1
```

系統就知道：

```text
2026/08/22 22:00
      ↓
2026/08/23 06:00
```

---

# 3. `shift_breaks`

## 說明

**班別休息時間。**

休息時間不能簡單用：

```text
工作 8 小時
```

來處理。

例如：

```text
08:00 → 12:00
12:00 → 13:00 休息
13:00 → 17:00
```

## Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 休息時間 ID |
| `shift_id` | `uuid` | 班別 ID |
| `sequence` | `integer` | 順序 |
| `start_time` | `time` | 休息開始 |
| `end_time` | `time` | 休息結束 |
| `start_day_offset` | `integer` | 開始日偏移 |
| `end_day_offset` | `integer` | 結束日偏移 |
| `break_type_code` | `integer` | 休息類型 |
| `is_paid` | `boolean` | 是否計薪 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

這裡特別保留 `is_paid`。

因為未來不同公司的休息規則可能不同。

---

# 4. `shift_rules`

這張是為了**最大化彈性**。

## 說明

班別的判定規則。

例如：

```text
08:00 上班
允許 08:10 前打卡
超過才算遲到

17:00 下班
允許提前 10 分鐘
```

或者：

```text
彈性上下班
一天工作滿 8 小時即可
```

## Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 班別規則 ID |
| `shift_id` | `uuid` | 班別 ID |
| `late_rule_code` | `integer` | 遲到判定規則 |
| `early_leave_rule_code` | `integer` | 早退判定規則 |
| `overtime_rule_code` | `integer` | 加班判定規則 |
| `rounding_rule_code` | `integer` | 打卡時間取整規則 |
| `minimum_work_minutes` | `integer` | 最低工作分鐘 |
| `maximum_work_minutes` | `integer` | 最大工作分鐘 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 5. `employee_schedules`

這才是真正的**員工班表**。

## 說明

指定：

> 某員工某一天要上哪個班。

例如：

```text
王小明

2026/08/24 → 一般班
2026/08/25 → 一般班
2026/08/26 → 休息
2026/08/27 → 夜班
```

## Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 班表 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `shift_id` | `uuid` | 班別 ID，可 NULL |
| `work_date` | `date` | 工作日期 |
| `schedule_type_code` | `integer` | 班表類型 |
| `status_code` | `integer` | 班表狀態 |
| `source_type_code` | `integer` | 班表來源 |
| `note` | `string` | 備註 |
| `created_by` | `uuid` | 建立人員 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 6. 為什麼 `shift_id` 可以 NULL？

因為：

```text
2026/08/26
休假
```

不是「沒有班表」。

而是：

> **這一天有排班，但排的是休息。**

所以我會用：

```text
shift_id = NULL
schedule_type_code = 2
```

例如：

```text
1 = 工作
2 = 休息
3 = 例假
4 = 休假
5 = 特殊
```

這些都只是 code，不使用 enum。

---

# 7. `schedule_rules`

這是最大化設計裡非常重要的一張。

因為不能要求管理者每天手動建立：

```text
8/1 一般班
8/2 一般班
8/3 一般班
...
```

應該可以建立週期。

## 說明

**員工週期性班表規則。**

例如：

```text
週一～週五
一般班

週六、週日
休息
```

或者：

```text
四休二
A A A A R R
```

## Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 班表規則 ID |
| `company_id` | `uuid` | 公司 ID |
| `name` | `string` | 規則名稱 |
| `cycle_type_code` | `integer` | 週期類型 |
| `cycle_length` | `integer` | 週期長度 |
| `start_date` | `date` | 規則開始日期 |
| `end_date` | `date` | 規則結束日期 |
| `is_active` | `boolean` | 是否啟用 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 8. `schedule_rule_details`

## 說明

**班表週期的實際內容。**

例如：

```text
四休二
Day 1 → A班
Day 2 → A班
Day 3 → A班
Day 4 → A班
Day 5 → 休息
Day 6 → 休息
```

## Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 規則明細 ID |
| `schedule_rule_id` | `uuid` | 班表規則 ID |
| `cycle_day` | `integer` | 週期第幾天 |
| `shift_id` | `uuid` | 班別 ID，可 NULL |
| `schedule_type_code` | `integer` | 當日班表類型 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 9. `employee_schedule_rules`

這張負責：

> **哪個員工套用哪個週期班表。**

## Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工班表規則 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `schedule_rule_id` | `uuid` | 班表規則 ID |
| `start_date` | `date` | 套用開始日期 |
| `end_date` | `date` | 套用結束日期 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 10. 為什麼還需要 `employee_schedules`？

因為：

> **週期規則 ≠ 最終班表。**

例如：

```text
週一～週五 → 一般班
```

原本如此。

但主管臨時安排：

```text
8/27 改成早班
```

這時不能修改整個週期規則。

所以：

```text
schedule_rule
        ↓
產生 employee_schedules
        ↓
8/27 個別調整
```

這樣非常重要。

---

# 11. `holiday_calendars`

如果要最大化，我建議現在就把假日獨立出來。

## 說明

**公司使用的假日行事曆。**

例如：

```text
2026 台灣行事曆
2027 台灣行事曆
公司自訂行事曆
```

## Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 假日行事曆 ID |
| `company_id` | `uuid` | 公司 ID |
| `name` | `string` | 行事曆名稱 |
| `year` | `integer` | 年度 |
| `is_active` | `boolean` | 是否啟用 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 12. `holiday_calendar_dates`

## 說明

**實際假日日期。**

例如：

```text
2026/01/01 元旦
2026/02/17 春節
2026/02/18 春節
...
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 假日日期 ID |
| `holiday_calendar_id` | `uuid` | 行事曆 ID |
| `holiday_date` | `date` | 日期 |
| `name` | `string` | 假日名稱 |
| `holiday_type_code` | `integer` | 假日類型 |
| `is_working_day` | `boolean` | 是否為工作日 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

這裡的 `is_working_day` 很重要。

因為台灣有：

> 連假補班。

所以不能簡單認為：

```text
星期六 = 一定休息
```

---

# 13. `employee_holiday_calendars`

## 說明

指定員工使用哪個假日行事曆。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工假日設定 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `holiday_calendar_id` | `uuid` | 假日行事曆 ID |
| `start_date` | `date` | 生效日期 |
| `end_date` | `date` | 結束日期 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 14. 最終完整關係

```text
                     shift_definitions
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
      shift_work_periods            shift_breaks
              │
              ▼
         shift_rules
              │
              ▼
       schedule_rules
              │
              ▼
    schedule_rule_details
              │
              ▼
  employee_schedule_rules
              │
              ▼
      employee_schedules
              │
              ▼
      attendance_records
```

另外：

```text
holiday_calendars
        │
        ▼
holiday_calendar_dates
        │
        ▼
employee_holiday_calendars
        │
        ▼
employee_schedules
```

---

# 15. 最重要的資料流

例如王小明：

### 公司建立班別

```text
一般班
08:00 ～ 17:00
12:00 ～ 13:00 休息
```

↓

### 建立週期

```text
週一～週五 → 一般班
週六 → 休息
週日 → 休息
```

↓

### 王小明套用

```text
2026/08/01 ～ 2026/12/31
套用「一般行政班」
```

↓

### 系統產生實際班表

```text
8/24 → 一般班
8/25 → 一般班
8/26 → 一般班
8/27 → 一般班
8/28 → 一般班
8/29 → 休息
8/30 → 休息
```

↓

### 臨時調班

主管：

```text
8/27
改成早班
```

只修改：

```text
employee_schedules
```

不修改：

```text
schedule_rules
```

↓

### 員工打卡

```text
08:07 上班
17:03 下班
```

↓

### 出勤計算

```text
應上班：08:00
實際：08:07
→ 遲到 7 分鐘

應下班：17:00
實際：17:03
→ 無早退
```

---

# 16. 我會再加一層：`attendance_results`

這張**不是你現在問的打卡紀錄**，但如果最大化設計，我建議預留。

因為：

```text
attendance_records
```

只記錄：

> **「員工做了什麼」**

而：

```text
attendance_results
```

記錄：

> **「系統判定結果是什麼」**

例如：

```text
應上班 08:00
實際上班 08:07
遲到 7 分鐘

應下班 17:00
實際下班 17:03

應工作 480 分鐘
實際工作 476 分鐘
```

這樣未來請假、加班、薪資計算才不會把「原始資料」和「計算結果」混在一起。

---

## 我最推薦的最終分層

```text
【定義層】

shift_definitions
shift_work_periods
shift_breaks
shift_rules


【排班層】

schedule_rules
schedule_rule_details
employee_schedule_rules
employee_schedules


【假日層】

holiday_calendars
holiday_calendar_dates
employee_holiday_calendars


【實際出勤層】

attendance_records


【計算結果層】

attendance_results


【補登層】

attendance_correction_requests
```

這樣設計的好處是，未來即使遇到：

- 一般行政班
- 輪班
- 四休二
- 三班制
- 夜班跨日
- 彈性上下班
- 多段工作時間
- 多段休息
- 補班
- 國定假日
- 公司自訂假日
- 臨時調班
- 員工個別調班
- 忘記上班打卡
- 忘記下班打卡
- 誤打撤銷
- GPS 打卡
- Web / 手機 / 打卡機
- 遲到、早退
- 加班

都還有位置可以放，而不需要把 `attendance_records` 做成一張「什麼都塞進去」的超大表。

**這一版我會建議先不要急著記錄。** 我們可以先檢查這套「最大化班表設計」有哪些可能的坑，再決定哪些 Schema 正式納入。

---

## 2026/08/22 21:51:32

### 使用者

employee_holiday_calendars    很奇怪

另外打零工的那種了
且擬剛才還有提到做二休二那種，一周只有7天，那你要怎麼設計

### 助理

對，你抓到兩個很重要的問題。前一版的設計確實有點「把傳統週一～週五公司」想得太重了。

尤其是：

1. `employee_holiday_calendars` 的確很奇怪，**假日不應該直接綁員工**
2. `二休二、四休二、三班輪班`根本不是「一週」的概念
3. 零工、兼職、臨時工可能**根本沒有固定班表**
4. 所以班表核心應該從「星期」改成**日期 + 週期**。

我重新整理一版。

---

# 一、先把「假日」拿掉員工綁定

我同意你。

原本：

```text
holiday_calendars
        ↓
employee_holiday_calendars
        ↓
employee
```

我不建議。

因為實際上應該是：

```text
公司
 ↓
工作日 / 假日規則
 ↓
班表
```

而不是：

```text
員工
 ↓
假日行事曆
```

而且很多輪班人員：

> 國定假日可能照樣上班。

所以「假日」不能直接等於「這個員工不上班」。

---

# 二、重新定義：班表才是最終答案

我們真正要問的是：

> **這個員工在某一天，應不應該工作？如果要工作，要上什麼班？**

所以最後一定要落到：

```text
employee_schedules
```

例如：

| 日期 | 員工 | 班別 |
|---|---|---|
| 8/24 | 王小明 | 一般班 |
| 8/25 | 王小明 | 一般班 |
| 8/26 | 王小明 | 休息 |
| 8/27 | 王小明 | 早班 |

這張才是**最終事實**。

---

# 三、二休二怎麼辦？

這就是前面設計最需要修正的地方。

假設：

```text
上班 2 天
休息 2 天
```

不是：

```text
星期一
星期二
星期三
...
```

而是：

```text
Day 1 → 工作
Day 2 → 工作
Day 3 → 休息
Day 4 → 休息
```

然後循環：

```text
Day 5 → 工作
Day 6 → 工作
Day 7 → 休息
Day 8 → 休息
```

所以應該叫：

# `schedule_cycles`

而不是把它理解成「每週班表」。

---

# 四、`schedule_cycles`

### 說明

**班表週期定義。**

週期可以是：

```text
5天
7天
8天
12天
14天
28天
甚至 30 天
```

完全不需要跟星期對齊。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 班表週期 ID |
| `company_id` | `uuid` | 公司 ID |
| `code` | `string` | 週期代碼 |
| `name` | `string` | 週期名稱 |
| `cycle_length` | `integer` | 週期天數 |
| `description` | `string` | 說明 |
| `is_active` | `boolean` | 是否啟用 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 五、`schedule_cycle_details`

### 說明

定義週期中的每一天要做什麼。

例如：

## 二休二

```text
schedule_cycles

cycle_length = 4
```

明細：

| cycle_day | schedule_type_code | shift |
|---:|---:|---|
| 1 | 1 工作 | A班 |
| 2 | 1 工作 | A班 |
| 3 | 2 休息 | NULL |
| 4 | 2 休息 | NULL |

---

## 四休二

```text
cycle_length = 6
```

| cycle_day | 類型 | 班別 |
|---:|---|---|
| 1 | 工作 | A班 |
| 2 | 工作 | A班 |
| 3 | 工作 | A班 |
| 4 | 工作 | A班 |
| 5 | 休息 | NULL |
| 6 | 休息 | NULL |

---

## 三班輪班

甚至可以：

```text
早
早
早
休
晚
晚
晚
休
夜
夜
夜
休
```

`cycle_length = 12`

完全沒問題。

---

# 六、那員工怎麼套用？

再建立：

# `employee_schedule_cycles`

### 說明

表示：

> **這個員工從哪一天開始套用哪一個週期。**

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工週期班表 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `schedule_cycle_id` | `uuid` | 班表週期 ID |
| `start_date` | `date` | 週期起始日期 |
| `end_date` | `date` | 週期結束日期，可 NULL |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

這個 `start_date` **非常重要**。

---

# 七、為什麼一定要有 `start_date`？

假設王小明：

```text
2026/08/01
開始二休二
```

那麼：

```text
8/1 → Day 1 工作
8/2 → Day 2 工作
8/3 → Day 3 休
8/4 → Day 4 休
8/5 → Day 1 工作
8/6 → Day 2 工作
...
```

系統計算：

```text
cycle_day =
(日期 - start_date) % cycle_length
```

所以它完全不需要星期。

---

# 八、這也解決「零工」

這個才是更重要的。

零工可能是：

```text
8/1 上班
8/2 不上
8/3 上班
8/4 上班
8/5 不上
8/6 不上
```

他根本沒有：

```text
週一～週五
```

也沒有：

```text
二休二
```

所以他可以**不套任何 cycle**。

直接建立：

```text
employee_schedules
```

例如：

| 日期 | 班別 |
|---|---|
| 8/1 | 早班 |
| 8/3 | 晚班 |
| 8/4 | 一般班 |

這就是：

> **個別排班**

---

# 九、所以我們其實需要兩種排班來源

### ① 週期排班

```text
employee_schedule_cycles
        ↓
schedule_cycles
        ↓
schedule_cycle_details
```

適合：

- 二休二
- 四休二
- 三班制
- 四班二輪
- 固定週一～週五
- 輪班

---

### ② 個別排班

```text
employee_schedules
```

適合：

- 零工
- 兼職
- 臨時工
- 臨時調班
- 特殊工作安排
- 主管臨時指定某一天上班

---

# 十、但有一個問題：週期排班產生後，要不要真的寫入 `employee_schedules`？

我的答案是：

## 要。

例如：

```text
employee_schedule_cycles

王小明
二休二
2026/08/01 開始
```

系統可以產生：

```text
employee_schedules

8/1 工作 A班
8/2 工作 A班
8/3 休息
8/4 休息
8/5 工作 A班
8/6 工作 A班
...
```

這樣真正的出勤系統只需要看：

```text
employee_schedules
```

不用每次重新計算週期。

---

# 十一、臨時調班就非常簡單

原本：

```text
8/5
A班
```

主管說：

> 8/5 改晚班。

直接修改：

```text
employee_schedules

8/5 → 晚班
```

**不用修改二休二週期。**

這就是為什麼：

```text
週期規則
```

和：

```text
實際班表
```

必須分開。

---

# 十二、那國定假日怎麼辦？

這時候我反而建議**不要建立 `employee_holiday_calendars`**。

只建立公司層級：

### `holiday_calendars`

```text
公司
 ↓
holiday_calendars
 ↓
holiday_calendar_dates
```

但是它只代表：

> 今天是什麼性質的日期。

例如：

```text
8/15
中秋節
holiday_type_code = 1
```

**它不直接決定員工不上班。**

最後還是由：

```text
employee_schedules
```

決定這個人到底有沒有排班。

---

# 十三、這樣就能處理最麻煩的情況

例如：

```text
8/15
中秋節
```

王小明：

```text
employee_schedules
8/15 → 休息
```

所以：

> 國定假日 + 沒排班

另一個輪班員工：

```text
employee_schedules
8/15 → 夜班
```

所以：

> 國定假日 + 有排班

系統就可以依公司的出勤與薪資規則，再判斷：

- 正常出勤
- 國定假日出勤
- 加班
- 工資倍率

**假日資料本身不直接干涉班表。**

這個架構我認為比前一版正確很多。

---

# 十四、我會把目前班表架構改成這樣

```text
                    ┌──────────────────┐
                    │  shift_definitions│
                    └────────┬─────────┘
                             │
                 ┌───────────┴───────────┐
                 ▼                       ▼
       shift_work_periods          shift_breaks
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
       employee_schedules
                ▲
                │
       個別/臨時排班直接建立


       holiday_calendars
                │
                ▼
       holiday_calendar_dates
```

然後：

```text
employee_schedules
        ↓
attendance_records
        ↓
attendance_results
        ↓
薪資
```

---

## 最核心的設計原則

我現在會把它定義成：

> **「週」只是其中一種週期，不是班表的基礎單位。**

班表真正的基礎單位是：

```text
日期
```

週期只是**產生日期班表的一種工具**。

所以：

```text
固定週休二日       → 7 天週期
二休二             → 4 天週期
四休二             → 6 天週期
三班輪班           → 任意週期
零工               → 無週期，直接排日期
臨時工             → 無週期或個別日期
```

這樣才是真的「最大化」，而且不會被「一週只有 7 天」這個概念綁死。

---

## 2026/08/22 21:55:26

### 使用者

我的想法還有一種，就是只要是她在她的班表體系中，定義是假日上班的，才可以申請加班，那種種擬怎麼設計

### 助理

對，這個想法我認為**應該直接放進班表體系**，而不是等到加班申請時才去猜「今天是不是假日」。

你要表達的其實是：

> **某員工在某一天的排班定義中，如果這一天被定義為「假日出勤」，才具備申請假日加班的資格。**

這跟「今天是不是國定假日」是兩回事。

---

# 一、先區分三個概念

例如 8/15 是公司定義的國定假日。

### 員工 A

班表：

```text
8/15 → 休息
```

代表：

```text
國定假日
+ 沒有排班
```

他不能因為自己跑來公司就直接申請「假日加班」。

---

### 員工 B

班表：

```text
8/15 → 假日上班
```

代表：

```text
國定假日
+ 班表明確安排出勤
```

他才有資格申請這一天的假日加班。

---

### 員工 C

班表：

```text
8/15 → 一般工作日
```

例如他的輪班制度根本沒有把 8/15 視為休假。

那就不能單純因為：

```text
holiday_calendar_dates = 國定假日
```

就直接判定為假日加班。

---

# 二、所以我建議 `employee_schedules` 增加「出勤性質」

目前我們有：

```text
employee_schedules
```

我會加入：

```text
schedule_day_type_code
```

### `schedule_day_type_code`

例如：

```text
1 = 工作日
2 = 休息日
3 = 例假日
4 = 國定假日
5 = 特別休假日
6 = 假日出勤
7 = 其他
```

但這裡我會稍微修正：

**不要把「國定假日」跟「假日出勤」完全混成同一層。**

更好的設計是拆成兩個概念。

---

# 三、`schedule_day_type_code`

表示：

> **這一天在這個員工的班表中是什麼性質。**

例如：

```text
1 = 工作日
2 = 休息日
3 = 例假日
4 = 假日
```

---

# 四、`scheduled_work_flag`

另外：

```text
scheduled_work_flag
```

表示：

> **這一天是否被安排出勤。**

例如：

| 日期 | Day Type | 出勤 |
|---|---|---|
| 8/14 | 工作日 | 是 |
| 8/15 | 假日 | 是 |
| 8/16 | 休息日 | 否 |
| 8/17 | 例假日 | 否 |

這樣：

```text
8/15
schedule_day_type_code = 4
scheduled_work_flag = true
```

就可以代表：

> **假日出勤**

而不是額外創造一個「假日出勤」類型。

我比較推薦這個方式。

---

# 五、但是「能不能申請加班」也不應該只靠這兩個欄位

因為未來可能有：

```text
假日可以加班
休息日可以加班
工作日下班後可以加班
```

所以我會再加入：

```text
overtime_eligibility_code
```

### 例如

```text
1 = 不可申請加班
2 = 可申請一般加班
3 = 可申請休息日加班
4 = 可申請假日加班
5 = 依公司規則
```

---

# 六、因此 `employee_schedules` 可以變成

### `employee_schedules`

**說明：**

> 員工某一日期的最終排班結果，是出勤系統判定的核心資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 班表 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `work_date` | `date` | 工作日期 |
| `shift_id` | `uuid` | 班別 ID，可 NULL |
| `schedule_day_type_code` | `integer` | 當日班表性質 |
| `scheduled_work_flag` | `boolean` | 是否安排出勤 |
| `overtime_eligibility_code` | `integer` | 加班申請資格 |
| `source_type_code` | `integer` | 班表來源 |
| `note` | `string` | 備註 |
| `created_by` | `uuid` | 建立人員 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 七、這樣輪班就很好處理

例如「二休二」。

週期：

```text
Day 1 → 工作日
Day 2 → 工作日
Day 3 → 休息日
Day 4 → 休息日
```

如果剛好遇到國定假日：

```text
Day 1
```

公司規則認定這個員工這天需要上班。

產生的實際班表：

```text
work_date = 2026/08/15

schedule_day_type_code = 4
scheduled_work_flag = true
overtime_eligibility_code = 4
shift_id = 夜班
```

這個人就具備：

> **假日加班申請資格。**

---

# 八、但我會再做一個很重要的設計

你剛剛說：

> 「只要是她在她的班表體系中，定義是假日上班的，才可以申請加班」

那代表**加班資格應該在「班表規則」產生時就決定**。

所以：

```text
schedule_cycles
        ↓
schedule_cycle_details
        ↓
employee_schedule_cycles
        ↓
employee_schedules
```

其中：

### `schedule_cycle_details`

也應該有：

| 欄位 | 型態 | 說明 |
|---|---|---|
| `schedule_day_type_code` | `integer` | 這一天的班表性質 |
| `scheduled_work_flag` | `boolean` | 是否出勤 |
| `overtime_eligibility_code` | `integer` | 加班資格 |

例如「二休二」：

```text
Day 1
工作日
出勤 = true
加班 = 一般加班

Day 2
工作日
出勤 = true
加班 = 一般加班

Day 3
休息日
出勤 = false
加班 = 不可

Day 4
休息日
出勤 = false
加班 = 不可
```

---

# 九、這會讓「假日上班」非常清楚

例如公司有一個輪班體系：

```text
A班
```

規則：

```text
正常工作日 → 可以正常工作
休息日 → 不排班
國定假日 → 如果排班，視為假日出勤
```

那產生班表時：

```text
8/15
```

如果被排到：

```text
schedule_day_type_code = 4
scheduled_work_flag = true
overtime_eligibility_code = 4
```

---

# 十、加班申請就不用自己判斷

未來建立：

```text
overtime_requests
```

員工申請：

```text
2026/08/15
18:00 ~ 22:00
```

系統直接找：

```text
employee_schedules
```

得到：

```text
overtime_eligibility_code = 4
```

所以：

```text
允許申請
```

如果：

```text
overtime_eligibility_code = 1
```

直接：

```text
不允許申請
```

這比在 `overtime_requests` 裡面自己判斷「今天是不是星期日、是不是國定假日」可靠很多。

---

# 十一、還有一個我認為很重要的點

**「可以申請加班」≠「已經加班」。**

所以之後一定要分：

```text
班表
 ↓
加班資格
 ↓
員工申請
 ↓
主管審核
 ↓
實際加班
 ↓
加班費
```

例如：

```text
8/15
班表：假日出勤
↓
具備假日加班資格

員工申請
18:00～22:00
↓
主管核准

實際打卡
18:05～21:50
↓
實際加班 3 小時 45 分
```

最後薪資才根據**實際核准的加班資料**計算。

---

## 我會把這個概念正式定義成

```text
【Holiday】
今天是不是法定/公司假日
        ↓
【Schedule】
這個員工今天有沒有被排班
        ↓
【Overtime Eligibility】
這個員工今天具不具備哪種加班申請資格
        ↓
【Overtime Request】
員工實際申請多少加班
        ↓
【Approval】
主管是否核准
        ↓
【Actual Attendance】
實際打卡多少
        ↓
【Payroll】
最後算多少加班費
```

這樣就能同時支援你說的**「輪班員工只有在他的班表體系被定義為假日出勤時，才可以申請假日加班」**，也不會把「國定假日」直接等同於「所有員工都放假」。

---

## 2026/08/22 21:57:35

### 使用者

employee_schedules   擬這個會是什麼時候產生?

### 助理

我會建議：**`employee_schedules` 是「實際班表」，在排班確定時產生，不是在員工打卡時才產生。**

而且這張表我會把它定位得很重要：

> **它是某個員工「某一天到底被安排什麼班」的最終快照。**

---

# 什麼時候產生？

主要有 4 種情況。

### ① 套用週期班表時

例如公司設定：

```text
二休二
Day 1 → A班
Day 2 → A班
Day 3 → 休息
Day 4 → 休息
```

王小明從：

```text
2026/08/01
```

開始套用。

系統就可以產生：

```text
employee_schedules

08/01 → A班
08/02 → A班
08/03 → 休息
08/04 → 休息
08/05 → A班
08/06 → A班
...
```

---

### ② 管理者排班時

例如零工：

```text
王小明

8/25 → 早班
8/27 → 晚班
8/30 → 一般班
```

因為他沒有固定週期，所以直接建立：

```text
employee_schedules
```

---

### ③ 臨時調班時

原本：

```text
8/25 → 一般班
```

主管修改：

```text
8/25 → 早班
```

這時修改 `employee_schedules`。

**不應該去修改原本的週期規則。**

---

### ④ 假日 / 補班等特殊情況

例如：

```text
8/15
國定假日
```

某輪班員工：

```text
employee_schedules

8/15
夜班
schedule_day_type = 假日
scheduled_work = true
overtime_eligibility = 假日加班
```

這個結果也應該在班表產生時確定。

---

# 為什麼不要等到打卡時才產生？

例如：

```text
8/15
```

員工來打卡：

```text
09:00
```

系統才開始問：

> 他今天有沒有班？

這樣會有很多問題。

因為你可能需要知道：

```text
他今天原本應該上什麼班？
今天是不是假日？
今天是不是休息日？
今天能不能申請加班？
應該幾點上班？
應該幾點下班？
```

這些都應該在**排班階段就決定**。

---

# 我更推薦「提前產生」

例如公司每月排下個月班表：

```text
8/20
```

管理者建立：

```text
9月份班表
```

系統產生：

```text
employee_schedules
9/1
9/2
9/3
...
9/30
```

之後員工才能看到：

> 我 9 月哪天上班、哪天休息。

---

# 但是要注意：不要無限產生

我不建議：

```text
一次產生未來 10 年
```

比較合理的是：

> **產生到公司目前已排定的期間。**

例如：

```text
目前日期：8/22

公司排班期間：
9/1 ～ 9/30

→ 產生 9 月 employee_schedules
```

如果公司只排到：

```text
9/15
```

就只產生到 9/15。

---

# 還有一個非常重要的問題

我會建議：

### `employee_schedules` 不只是「系統計算結果」

而是：

> **「已確定的實際班表」**

所以一旦產生：

```text
employee_schedules
```

它就應該可以被管理者個別修改。

例如週期規則：

```text
二休二
```

產生：

```text
8/25 → A班
```

主管臨時改：

```text
8/25 → B班
```

那就直接修改這一天的：

```text
employee_schedules
```

而不是重新修改週期。

---

# 我會再加一個欄位

因為未來你一定會遇到：

> 「這個班表為什麼會變成這樣？」

所以 `employee_schedules` 我會加入：

```text
source_type_code
```

例如：

```text
1 = 週期產生
2 = 管理者排班
3 = 臨時調班
4 = 系統產生
```

甚至可以再加：

```text
source_id
```

用來追溯：

> 這筆班表是從哪個週期規則產生的。

---

# 所以完整流程會是

```text
班別
 ↓
班表週期
 ↓
員工套用週期
 ↓
產生 employee_schedules
 ↓
管理者可以調整
 ↓
班表確定
 ↓
員工打卡
 ↓
attendance_records
 ↓
attendance_results
 ↓
加班 / 請假 / 薪資
```

### 最核心的一句話：

**`employee_schedules` 是「排班結果」，不是「排班規則」。**

所以它應該在**班表被建立/發布時產生**，而不是等員工打卡時才產生。

另外，我會建議下一步討論一個很關鍵的東西：**「班表發布 / 鎖定」**。:chatgpt-content-reference{index="0"}。這會直接影響 `employee_schedules` 的設計。

---

## 2026/08/22 21:57:59

### 使用者

那擬這樣會排多久

### 助理

如果採用剛才的設計，我會建議不要固定「一次排一年」或「一次排一個月」，而是把**排班期間**交給公司決定。

但從系統設計角度，我會預設：

> **至少提前產生 1 個完整的排班週期，通常建議預排 1～3 個月。**

### 不同類型

| 類型 | 建議預排 |
|---|---|
| 一般週休二日 | 1～3 個月 |
| 二休二 / 四休二 | 1～3 個月 |
| 三班輪班 | 1～3 個月 |
| 零工 / 兼職 | 依實際排班，可能只排 1～2 週 |
| 臨時工 | 有需要才建立 |
| 特殊大型排班 | 公司自行決定 |

例如現在是 **8/22**：

公司決定排到 10/31：

```text
employee_schedules

8/1 ～ 10/31
```

之後到了 9 月，再繼續排 11 月、12 月。

---

## 但我更建議增加「排班期間」

不要讓 `employee_schedules` 自己猜。

可以有：

### `schedule_periods`

說明：

> 公司某一次正式排班的期間。

例如：

```text
2026/09/01 ～ 2026/09/30
2026/10/01 ～ 2026/10/31
```

Schema：

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 排班期間 ID |
| `company_id` | `uuid` | 公司 ID |
| `name` | `string` | 排班期間名稱 |
| `start_date` | `date` | 開始日期 |
| `end_date` | `date` | 結束日期 |
| `status_code` | `integer` | 排班期間狀態 |
| `published_at` | `datetime` | 發布時間 |
| `published_by` | `uuid` | 發布人員 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

然後：

```text
schedule_periods
       ↓
employee_schedules
```

`employee_schedules` 加：

```text
schedule_period_id
```

這樣就非常清楚：

> **這筆班表屬於哪一次排班。**

---

### 我會推薦的流程

```text
建立 9 月排班期間
        ↓
套用二休二 / 固定班 / 個別排班
        ↓
產生 employee_schedules
        ↓
主管調整
        ↓
確認
        ↓
發布
        ↓
員工看到班表
```

而且**發布前可以一直修改**。

發布後如果要改：

```text
原班表
   ↓
臨時調班
   ↓
留下修改紀錄
```

這樣會比單純「一直修改 `employee_schedules`」安全很多。

所以我會把**「排班期間 `schedule_periods`」納入正式 Schema**。它能解決你剛才問的「到底排多久」以及後面的「哪一次排班、是否已發布、發布後能不能修改」問題。

