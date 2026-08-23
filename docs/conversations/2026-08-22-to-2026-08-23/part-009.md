# 人資系統需求原始對話紀錄 — Part 009

> 來源：ChatGPT 對話「整理人資系統需求」  
> 時區：Asia/Taipei  
> 範圍：2026/08/22 22:25:09 ～ 2026/08/22 22:39:33  
> 說明：保留原始對話順序，未去除重複、舊版或後續已修正內容。

---

## 2026/08/22 22:25:09

### 使用者

勞基法的假別規定先出來

### 助理

可以。我們現在先**不要設計 Schema**，先把「台灣勞基法／相關法規下，公司 HR 系統需要支援哪些假別」整理清楚。

以下以 **2026 年目前勞動部公開資料**為基準。要注意：有些假別主要來自《勞動基準法》及《勞工請假規則》，有些則來自《性別平等工作法》；另外公司可以提供**優於法令的福利假**。勞動部也明確說明，《勞工請假規則》是最低標準，企業可以訂更優的規定。citeturn0search0turn0search1

# 一、先分成 4 大類

我建議我們不要把所有假都當成同一種東西。

```text
請假制度
│
├── A. 法定休假
│   ├── 例假
│   ├── 休息日
│   ├── 國定假日
│   └── 特別休假
│
├── B. 勞工請假規則
│   ├── 婚假
│   ├── 喪假
│   ├── 事假
│   ├── 普通傷病假
│   ├── 公傷病假
│   └── 公假
│
├── C. 性別平等工作法
│   ├── 生理假
│   ├── 產假
│   ├── 產檢假
│   ├── 陪產檢及陪產假
│   ├── 安胎休養請假
│   ├── 育嬰留職停薪
│   └── 家庭照顧假
│
└── D. 公司自訂
    ├── 公司贈與特休
    ├── 公司贈與補休
    ├── 福利假
    └── 其他公司福利假
```

不過其中 **例假、休息日、國定假日**其實比較接近「排班／休假日」，不完全是員工提出的「請假」。

所以我們後面做 Schema 時，我會把它們和真正的 `leave_request` 分開。

---

# 二、婚假

| 項目 | 法定規定 |
|---|---|
| 假別 | 婚假 |
| 日數 | **8 日** |
| 工資 | **照給** |

勞動部目前資料列明婚假 8 日，工資照給。citeturn0search19

所以系統要能知道：

```text
婚假
8 日
有薪
```

而不是單純：

```text
leave_type = 婚假
```

因為後面還需要管理「已用多少、剩多少、適用期間」。

---

# 三、喪假

目前法規依親屬關係區分：

| 親屬 | 日數 |
|---|---:|
| 父母、養父母、繼父母、配偶 | 8 日 |
| 祖父母、子女、配偶之父母、配偶之養父母／繼父母 | 6 日 |
| 曾祖父母、兄弟姊妹、配偶之祖父母 | 3 日 |

以上工資照給。citeturn0search19

### 這對我們 Schema 很重要

喪假不能只設：

```text
leave_type = 喪假
```

還需要：

```text
relationship_code
```

例如：

```text
1 = 父母
2 = 配偶
3 = 子女
...
```

而且這個代碼可以直接存在 Schema 的欄位註釋裡，就符合我們之前「不要為了這種固定選項另外建表」的設計原則。

---

# 四、事假

```text
一年內最多 14 日
```

而且：

> **可以不給工資。**

勞動部目前資料也是如此。citeturn0search19

因此：

```text
事假
最大額度：14日／年
薪資：無薪
```

---

# 五、普通傷病假

這個比較複雜，也是我們請假 Schema 很重要的一種。

目前勞動部資料：

### 未住院

```text
一年內合計不得超過 30 日
```

### 住院

```text
兩年內合計不得超過 1 年
```

而未住院與住院傷病假在兩年內合計不得超過 1 年。

癌症門診治療、懷孕期間安胎休養等特殊情況也有併入住院傷病假計算的規定。citeturn0search19

### 工資

普通傷病假一年內未超過 30 日的部分：

> **工資折半發給**

符合條件的勞保普通傷病給付未達工資半數時，雇主補足。citeturn0search19

---

# 六、公傷病假

如果因為：

> 職業災害造成失能、傷害或疾病

在治療、休養期間可以請公傷病假。

工資涉及《勞動基準法》第 59 條的原領工資補償規定。citeturn0search19

這個我們未來 Schema **不能簡化成普通病假**。

它需要記錄：

```text
是否職業災害
相關證明
起始時間
終止時間
```

---

# 七、公假

公假不是「固定幾天」。

而是：

> **依法令規定應給予公假。**

工資照給。citeturn0search19

因此這類假別比較適合設計成：

```text
公假
→ 有法令／事由依據
→ 依實際事件產生
```

而不是固定：

```text
annual_quota = 7
```

---

# 八、特別休假

這個是我們後面 Schema **最重要的一個模組之一**。

勞工工作年資滿一定期間，即享有特別休假。citeturn0search2

而且有一個非常重要的原則：

> **特休日期原則上由勞工排定。**

雇主不能任意限制員工只能在某些日期請特休。citeturn0search5

另外還涉及：

- 年資
- 特休年度
- 本年度取得
- 遞延
- 遞延至次年度
- 未休工資

所以特休**絕對不能只做一個 `annual_leave_days` 欄位**。

---

# 九、生理假

這個是《性別平等工作法》體系，不是單純《勞工請假規則》。

目前規定：

> 因生理日致工作有困難者，每月得請生理假 1 日。

全年請假未逾 3 日部分，不併入病假；其餘日數併入病假計算。

工資：

> **減半發給。** citeturn0search18turn0search8


這代表 Schema 必須能處理：

```text
每月額度
+
年度併入病假計算的規則
```

---

# 十、家庭照顧假

目前：

```text
一年 7 日
```

而且：

> **併入事假計算。**

工資：

> 得不給薪。citeturn0search18

所以系統不能單純把它當成：

```text
事假
```

因為員工畫面上仍然應該是：

```text
家庭照顧假
```

但在法規計算上：

```text
家庭照顧假 → 計入事假額度
```

這是一個很典型的：

> **顯示假別 ≠ 額度計算假別**

的情況。

這對我們後面的 Schema 非常重要。

---

# 十一、產假

目前規定包含：

### 分娩

```text
8 星期
```

### 妊娠 3 個月以上流產

```text
4 星期
```

### 妊娠 2 個月以上未滿 3 個月流產

```text
1 星期
```

### 妊娠未滿 2 個月流產

```text
5 日
```

相關工資依《勞動基準法》第 50 條辦理。citeturn0search18

所以「產假」不能只是：

```text
quota = 56 days
```

因為**事由不同，法定額度不同**。

---

# 十二、產檢假

目前：

```text
7 日
```

而且：

> **工資照給。** citeturn0search18


---

# 十三、陪產檢及陪產假

目前：

```text
7 日
```

適用於：

- 陪伴配偶妊娠產檢
- 配偶分娩

工資照給。citeturn0search18

---

# 十四、安胎休養請假

這個也要列進我們系統。

而且它不是普通病假。

勞動部特別指出，安胎休養請假屬其他法令規定的假別，不計入普通傷病假「一年內未超過 10 日」的相關不利處分計算。citeturn0search8

因此系統要把它當成**獨立假別**。

---

# 十五、育嬰留職停薪

這個我要特別標記：

> **它不是一般「請假」那種概念。**

它屬於「留職停薪」制度。

所以我們後面做：

```text
leave_requests
```

時，我不建議把它跟：

```text
特休 8 小時
事假 4 小時
病假 2 小時
```

放成完全相同的資料模型。

它比較接近：

```text
employee
 ↓
leave_of_absence
 ↓
employment status / employment history
```

這個之後應該獨立設計。

---

# 十六、國定假日

這個也不要做成員工「請假」。

內政部所定紀念日、節日、勞動節及其他中央主管機關指定應放假日，依法應休假；部分工時勞工也有相關權益。citeturn0search2turn0search6

所以應該是：

```text
Holiday Calendar
```

而不是：

```text
Leave Request
```

這跟我們之前班表設計其實可以很好地接起來。

---

# 十七、例假、休息日

這兩個也不是「請假」。

勞基法第 36 條規定：

> 勞工每 7 日中應有 2 日休息，其中 1 日為例假、1 日為休息日。citeturn0search1turn0search2

所以：

```text
例假
休息日
```

應該屬於：

> **班表／工時制度**

而不是 Leave Module。

這也符合我們之前設計做二休二、輪班班表的方向。

---

# 十八、因此我們真正要做的「請假制度」其實是這樣

我建議最後分成：

```text
                    人事休假制度
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
   工作休息日          一般請假          留職停薪
       │                 │                 │
   例假                 婚假             育嬰留停
   休息日               喪假
   國定假日             事假
                        病假
                        公傷病假
                        公假
                        特休
                        生理假
                        家庭照顧假
                        產假
                        產檢假
                        陪產檢及陪產假
                        安胎休養
```

---

## 我認為這裡最重要的設計觀念

**不要把「法定假別」直接等同於「一個固定額度」。**

因為實際上有：

```text
固定額度
    ↓
事假 14 日

依年資
    ↓
特休

依親屬關係
    ↓
喪假

依月份
    ↓
生理假

依事件
    ↓
婚假、產假、公假

依其他假別計算
    ↓
家庭照顧假 → 併入事假

依職業災害
    ↓
公傷病假
```

所以我們接下來設計 Schema 時，**核心應該是「假別規則引擎 + 假別額度／資格 + 請假申請」**，而不是做一張死板的 `leave_types` 表塞一堆 `days`。

這會直接影響我們下一步的 Schema 設計。

---

## 2026/08/22 22:26:42

### 使用者

設計Schema

### 助理

可以。這一版我會把「**請假制度**」正式落成 Schema，而且會特別按照我們剛才討論出的問題來設計：**法定假別不是單純一個固定天數**，有些依年資、有些依月份、有些依事件、有些還會併入其他假別計算。

另外我查了勞動部 2026 年最新資料後，有一個重要更新會直接影響 Schema：**家庭照顧假自 2026/1/1 起可以用「小時」為單位，一年 56 小時**；而且仍併入事假計算。citeturn0search0turn0search2

---

# 一、先定義整個請假模組

我建議不要做成一張大表，而是：

```text
leave_types
    │
    ├── leave_type_rules
    │
    ├── leave_entitlements
    │
    └── leave_balances
             │
             ▼
       leave_requests
             │
             ├── leave_request_details
             │
             ├── leave_request_approvals
             │
             └── leave_documents
```

另外：

```text
leave_balances
      │
      └── leave_balance_transactions
```

負責追蹤額度歷史。

---

# 二、`leave_types`

## 用途說明

定義公司系統中的「假別」。

這張表**只描述假別是什麼**，不直接把「14 天、7 天、每月 1 天」硬寫死在這裡。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 假別 ID |
| `code` | `varchar(30)` | 必填 | 假別代碼，例如 `ANNUAL`、`SICK` |
| `name` | `varchar(50)` | 必填 | 假別名稱 |
| `category_code` | `int` | 必填 | `1=法定假別`、`2=性平法假別`、`3=公司福利假`、`4=其他` |
| `is_paid` | `boolean` | 必填 | 是否一般原則為有薪 |
| `requires_balance` | `boolean` | 必填 | 是否需要管理額度 |
| `requires_approval` | `boolean` | 必填 | 是否需要主管／HR 審核 |
| `requires_document` | `boolean` | 必填 | 是否原則上需要證明文件 |
| `unit_code` | `int` | 必填 | `1=日`、`2=小時`、`3=分鐘` |
| `is_active` | `boolean` | 必填 | 是否啟用 |
| `sort_order` | `int` | 必填 | UI 顯示順序 |
| `description` | `text` | 選填 | 假別說明 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

### 設計理由

例如：

```text
特別休假
事假
普通傷病假
婚假
喪假
生理假
家庭照顧假
產假
產檢假
陪產檢及陪產假
公假
公傷病假
```

都只是「假別」。

真正的：

> 可以請多少？

放到下一張。

---

# 三、`leave_type_rules`

## 用途說明

定義**假別的計算規則**。

這張是整個請假制度最重要的 Schema。

因為我們已經確認：

> 假別不能全部用固定天數表示。

例如：

```text
事假
→ 一年 14 日

生理假
→ 每月 1 日

特休
→ 依年資

喪假
→ 依親屬關係

產假
→ 依事件／懷孕週數

家庭照顧假
→ 一年 56 小時
→ 且併入事假
```

這些都屬於 Rule。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 規則 ID |
| `leave_type_id` | `bigint` | 必填 | 假別 |
| `rule_type_code` | `int` | 必填 | 規則類型 |
| `calculation_type_code` | `int` | 必填 | 額度計算方式 |
| `period_type_code` | `int` | 選填 | `1=曆年`、`2=到職週年`、`3=月份`、`4=事件期間`、`5=無固定週期` |
| `quota_minutes` | `int` | 選填 | 固定額度 |
| `max_quota_minutes` | `int` | 選填 | 最大額度 |
| `reference_leave_type_id` | `bigint` | 選填 | 額度併入哪個假別 |
| `eligibility_rule` | `json` | 選填 | 資格判定規則 |
| `quota_rule` | `json` | 選填 | 額度計算規則 |
| `salary_rule` | `json` | 選填 | 薪資處理規則 |
| `document_rule` | `json` | 選填 | 證明文件規則 |
| `effective_from` | `date` | 必填 | 規則生效日 |
| `effective_to` | `date` | 選填 | 規則失效日 |
| `is_active` | `boolean` | 必填 | 是否啟用 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

---

## 設計理由

這裡我故意沒有做：

```text
annual_days
monthly_days
```

因為太死。

例如家庭照顧假目前已經可以：

```text
56 小時
```

而不是只能用 7 天。citeturn0search2

因此：

```text
quota_rule
```

可以保存：

```text
{
    "period": "year",
    "amount": 56,
    "unit": "hour"
}
```

而特休則可以：

```text
{
    "type": "seniority",
    "rules": [...]
}
```

---

# 四、`leave_entitlements`

## 用途說明

記錄**某一位員工實際取得的假別額度**。

這和 `leave_type_rules` 不一樣。

Rule 是：

> 法律規定可以有多少。

Entitlement 是：

> 這個員工實際被系統核發多少。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 額度 ID |
| `employee_id` | `bigint` | 必填 | 員工 |
| `leave_type_id` | `bigint` | 必填 | 假別 |
| `source_type_code` | `int` | 必填 | `1=法定核發`、`2=公司贈與`、`3=遞延`、`4=人工調整`、`5=其他` |
| `source_id` | `bigint` | 選填 | 來源資料 ID |
| `entitled_minutes` | `int` | 必填 | 核發額度 |
| `effective_from` | `date` | 必填 | 生效日 |
| `effective_to` | `date` | 選填 | 到期日 |
| `status_code` | `int` | 必填 | `1=有效`、`2=撤銷`、`3=結清` |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

### 設計理由

例如員工今年：

```text
特休 7 天
公司額外贈與 2 天
去年遞延 3 天
```

不是：

```text
特休 = 12 天
```

而是三筆額度來源。

這跟我們剛才設計補休的概念一致。

---

# 五、`leave_balances`

## 用途說明

提供目前假別額度的快速查詢。

它是**目前狀態**，不是歷史帳本。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | ID |
| `employee_id` | `bigint` | 必填 | 員工 |
| `leave_type_id` | `bigint` | 必填 | 假別 |
| `entitled_minutes` | `int` | 必填 | 總取得額度 |
| `reserved_minutes` | `int` | 必填 | 已申請但尚未核准的凍結額度 |
| `used_minutes` | `int` | 必填 | 已使用額度 |
| `expired_minutes` | `int` | 必填 | 已過期額度 |
| `remaining_minutes` | `int` | 必填 | 目前可用額度 |
| `updated_at` | `datetime` | 必填 | 最後更新時間 |

### 設計理由

例如：

```text
特休
總額：80h
凍結：8h
已用：24h
剩餘：48h
```

UI 不需要每次重算所有交易。

但真正的歷史依據仍然是：

```text
leave_entitlements
+
leave_balance_transactions
```

---

# 六、`leave_balance_transactions`

## 用途說明

假別額度的**完整帳本**。

跟前面補休的 `compensatory_leave_transactions` 是同一個核心概念。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 交易 ID |
| `leave_entitlement_id` | `bigint` | 必填 | 哪一批額度 |
| `transaction_type_code` | `int` | 必填 | `1=核發`、`2=預約`、`3=使用`、`4=取消返還`、`5=遞延`、`6=到期`、`7=撤銷`、`8=人工調整` |
| `minutes` | `int` | 必填 | 異動分鐘 |
| `reference_type` | `varchar(50)` | 選填 | 來源類型 |
| `reference_id` | `bigint` | 選填 | 來源 ID |
| `occurred_at` | `datetime` | 必填 | 發生時間 |
| `created_by` | `bigint` | 必填 | 執行者 |
| `reason` | `text` | 選填 | 原因 |
| `created_at` | `datetime` | 必填 | 建立時間 |

### 設計理由

例如：

```text
特休 80h
↓
使用 8h
↓
取消 8h
↓
再次使用 4h
```

完整歷史都保留。

---

# 七、`leave_requests`

## 用途說明

員工提出的**請假申請主表**。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 請假申請 ID |
| `employee_id` | `bigint` | 必填 | 申請員工 |
| `request_no` | `varchar(30)` | 必填 | 申請編號 |
| `reason` | `text` | 選填 | 請假原因 |
| `status_code` | `int` | 必填 | `1=草稿`、`2=待審核`、`3=核准`、`4=拒絕`、`5=撤回`、`6=取消` |
| `applied_at` | `datetime` | 必填 | 申請時間 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

### 設計理由

這裡**不放**：

```text
leave_type_id
start_at
end_at
```

因為我們已經決定：

> **一張請假單可以混用不同假別。**

---

# 八、`leave_request_details`

## 用途說明

真正記錄「請什麼假、什麼時間、多少時間」。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 明細 ID |
| `leave_request_id` | `bigint` | 必填 | 請假主單 |
| `leave_type_id` | `bigint` | 必填 | 假別 |
| `start_at` | `datetime` | 必填 | 請假開始 |
| `end_at` | `datetime` | 必填 | 請假結束 |
| `requested_minutes` | `int` | 必填 | 申請分鐘 |
| `approved_minutes` | `int` | 選填 | 核准分鐘 |
| `status_code` | `int` | 必填 | 明細狀態 |
| `reason` | `text` | 選填 | 該明細原因 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

### 設計理由

例如：

```text
請假單 #1001

明細 1：特休 4h
明細 2：補休 4h
```

這就是我們之前確認的：

> **請假可以混用不同假別。**

---

# 九、`leave_request_approvals`

## 用途說明

記錄請假的審核歷史。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 審核 ID |
| `leave_request_id` | `bigint` | 必填 | 請假單 |
| `action_code` | `int` | 必填 | `1=送出`、`2=核准`、`3=拒絕`、`4=撤回`、`5=取消` |
| `action_by` | `bigint` | 必填 | 操作者 |
| `action_at` | `datetime` | 必填 | 時間 |
| `reason` | `text` | 選填 | 原因 |
| `created_at` | `datetime` | 必填 | 建立時間 |

### 設計理由

不能只：

```text
leave_requests.status = 3
```

就把之前的拒絕、重新送審全部抹掉。

---

# 十、`leave_request_allocations`

這張非常重要。

## 用途說明

記錄一筆請假明細**實際從哪一批假別額度扣除**。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | ID |
| `leave_request_detail_id` | `bigint` | 必填 | 請假明細 |
| `leave_entitlement_id` | `bigint` | 必填 | 使用哪一批額度 |
| `reserved_minutes` | `int` | 必填 | 凍結分鐘 |
| `used_minutes` | `int` | 必填 | 實際使用 |
| `released_minutes` | `int` | 必填 | 取消後返還 |
| `status_code` | `int` | 必填 | `1=預約`、`2=已使用`、`3=已釋放`、`4=取消` |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

### 設計理由

例如：

```text
特休 A：40h
特休 B：20h

請假 48h
```

可能：

```text
A → 40h
B → 8h
```

取消時：

```text
A → 原路返還 40h
B → 原路返還 8h
```

這跟我們前面設計補休的 `compensatory_leave_allocations` 完全一致。

---

# 十一、`leave_request_documents`

## 用途說明

記錄請假所需的證明文件。

因為勞動部目前規定，辦理請假手續時，雇主可以要求勞工提出相關證明文件。citeturn0search0turn0search6

所以不能直接把：

```text
medical_certificate
```

硬塞在 `leave_requests`。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 文件 ID |
| `leave_request_id` | `bigint` | 必填 | 請假單 |
| `document_type_code` | `int` | 必填 | `1=診斷證明`、`2=就醫證明`、`3=死亡證明`、`4=親屬關係證明`、`5=其他` |
| `file_id` | `bigint` | 必填 | 檔案 |
| `verified_at` | `datetime` | 選填 | 驗證時間 |
| `verified_by` | `bigint` | 選填 | 驗證者 |
| `status_code` | `int` | 必填 | `1=待驗證`、`2=通過`、`3=不通過` |
| `created_at` | `datetime` | 必填 | 建立時間 |

---

# 十二、`leave_events`

這張是我建議一定要有的。

## 用途說明

記錄「特殊事件」，讓系統可以根據事件決定員工可以請多少假。

因為：

```text
喪假
→ 親屬死亡

產假
→ 生產／流產

產檢假
→ 懷孕

陪產檢及陪產假
→ 配偶懷孕／分娩
```

這些不是單純的額度。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 事件 ID |
| `employee_id` | `bigint` | 必填 | 員工 |
| `event_type_code` | `int` | 必填 | `1=結婚`、`2=親屬死亡`、`3=分娩`、`4=流產`、`5=懷孕`、`6=配偶分娩`、`7=職業災害`、`8=其他` |
| `event_date` | `date` | 必填 | 事件日期 |
| `relationship_code` | `int` | 選填 | 親屬關係代碼 |
| `related_employee_id` | `bigint` | 選填 | 若事件涉及另一名員工 |
| `details` | `json` | 選填 | 事件詳細資料 |
| `document_verified` | `boolean` | 必填 | 是否完成證明驗證 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

### 設計理由

例如喪假：

```text
事件：
父親死亡

relationship_code = 1
```

系統才知道應套用哪一個喪假規則。

這比把：

```text
leave_request.relationship_code
```

當成唯一依據更乾淨。

---

# 十三、`special_leave_entitlements`

## 用途說明

特別休假獨立管理。

這是我刻意拆出來的，因為特休具有：

- 年資計算
- 特休年度
- 遞延
- 未休結清
- 可能跨年度
- 法定取得日數

這些邏輯比普通假別複雜很多。

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 特休額度 ID |
| `employee_id` | `bigint` | 必填 | 員工 |
| `service_years` | `decimal(4,2)` | 必填 | 計算時年資 |
| `entitlement_minutes` | `int` | 必填 | 法定取得額度 |
| `carry_forward_minutes` | `int` | 必填 | 前期遞延額度 |
| `effective_from` | `date` | 必填 | 生效日 |
| `effective_to` | `date` | 必填 | 使用期限 |
| `settlement_status_code` | `int` | 必填 | `1=未結算`、`2=已結算`、`3=遞延` |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

### 設計理由

特休不是單純：

```text
leave_entitlements
```

因為我們之後一定會需要處理：

```text
今年特休
去年遞延
到期未休
折算工資
```

所以單獨管理比較合理。

---

# 十四、育嬰留職停薪不要放進 `leave_requests`

這個我會直接定案：

```text
❌ leave_requests
```

而是下一個獨立模組：

```text
employee_leave_of_absences
```

因為它不是「請 8 小時假」。

例如育嬰留職停薪可能是：

```text
2026/09/01
~
2027/08/31
```

會影響員工的**任職狀態、薪資、福利、出勤、保險等**。

所以不能跟普通請假混在一起。

---

# 十五、最終關聯圖

```text
                         ┌──────────────────┐
                         │    leave_types   │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │ leave_type_rules │
                         └────────┬─────────┘
                                  │
                 ┌────────────────┴────────────────┐
                 ▼                                 ▼
       ┌──────────────────┐              ┌──────────────────┐
       │leave_entitlements│              │  leave_events    │
       └────────┬─────────┘              └──────────────────┘
                │
        ┌───────┴────────┐
        ▼                ▼
 leave_balances   leave_balance_transactions
        │
        ▼
 ┌────────────────┐
 │ leave_requests │
 └───────┬────────┘
         │
         ▼
┌────────────────────────┐
│ leave_request_details  │
└───────────┬────────────┘
            │
      ┌─────┴──────────┐
      ▼                ▼
allocations       documents
      │
      ▼
leave_entitlements
```

---

# 十六、這一版最重要的設計

我會把整個請假系統定義成：

```text
法規規則
    ↓
leave_type_rules
    ↓
員工符合資格
    ↓
leave_entitlements
    ↓
形成額度
    ↓
leave_balances
    ↓
員工申請
    ↓
leave_requests
    ↓
leave_request_details
    ↓
額度分配
    ↓
leave_request_allocations
    ↓
假別帳本
    ↓
leave_balance_transactions
```

而**特殊事件**則從旁邊進來：

```text
結婚
死亡
懷孕
生產
流產
職災
   ↓
leave_events
   ↓
leave_type_rules
   ↓
判斷可請假別與額度
```

這樣才能真正處理台灣目前的法規，而不是把 HR 系統做成「假別名稱 + 剩餘天數」而已。

尤其 2026 年開始，家庭照顧假已經可以以**小時**申請，普通傷病假也新增了「一年內未超過 10 日不得因此為不利處分」的規則；所以 `leave_type_rules` 必須能表達**額度規則與法規效果**，不能把規則全部硬編碼在 Schema 裡。citeturn0search0turn0search6

**下一個我建議不是馬上再加 Schema，而是:chatgpt-content-reference{index="5"}**，尤其是「特休、病假、家庭照顧假、喪假、公司福利假」這幾個最容易出問題的地方。

---

## 2026/08/22 22:27:14

### 使用者

剛剛我說的贈與特休跟贈與補休也要加入

### 助理

對，這兩個要正式加入，而且我建議**不要只靠 `source_type_code = 公司贈與` 帶過去**。

因為我們已經確定：

- 公司可以贈與員工 **特休**
- 公司可以贈與員工 **補休**
- 員工之間**不能互相轉贈**
- 公司贈與的額度仍然要有完整歷史
- 公司贈與的特休／補休可以有自己的生效日、到期日、原因與核發者

所以我會把它正式設計成一個共用的 **公司贈與額度 Schema**。

---

# 一、增加 `company_leave_grants`

## 用途說明

記錄「**公司核發／贈與給員工的假別額度**」。

這張表不是員工互轉。

資料流固定是：

```text
公司
 ↓
核發
 ↓
員工
 ↓
假別額度
```

可以贈與：

```text
特休
補休
其他公司自訂假
```

---

## Schema

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 公司贈與紀錄 ID |
| `employee_id` | `bigint` | 必填 | 被公司贈與的員工 |
| `leave_type_id` | `bigint` | 必填 | 贈與的假別，例如特休、補休 |
| `granted_minutes` | `int` | 必填 | 贈與分鐘數 |
| `effective_from` | `date` | 必填 | 額度生效日 |
| `effective_to` | `date` | 選填 | 額度到期日 |
| `reason` | `text` | 必填 | 贈與原因 |
| `status_code` | `int` | 必填 | `1=待核發`、`2=已核發`、`3=撤銷` |
| `granted_by` | `bigint` | 必填 | 有權限的核發者 |
| `granted_at` | `datetime` | 選填 | 實際核發時間 |
| `cancelled_by` | `bigint` | 選填 | 撤銷者 |
| `cancelled_at` | `datetime` | 選填 | 撤銷時間 |
| `cancel_reason` | `text` | 選填 | 撤銷原因 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

---

# 二、為什麼要獨立這張？

例如公司送員工：

```text
2026 年度福利
特休 +2 天
```

以及：

```text
專案獎勵
補休 +8 小時
```

這兩筆本質上都是：

> **公司贈與**

但後續進入不同的額度系統。

---

# 三、贈與特休

資料流：

```text
company_leave_grants
        │
        │ leave_type = 特休
        ▼
leave_entitlements
        │
        ▼
leave_balances
        │
        ▼
leave_requests
```

例如：

```text
公司贈與特休
+2 天
2026/09/01 ～ 2026/12/31
```

會產生一筆特休額度。

---

# 四、贈與補休

資料流：

```text
company_leave_grants
        │
        │ leave_type = 補休
        ▼
compensatory_leave_credits
        │
        ├── compensatory_leave_transactions
        └── compensatory_leave_allocations
```

例如：

```text
公司贈與補休
+8 小時
到期日：2026/12/31
```

產生一個獨立的補休批次。

---

# 五、這樣就能把「來源」分得很清楚

### 特休

```text
leave_entitlements
```

可能來源：

```text
1 = 法定取得
2 = 公司贈與
3 = 遞延
4 = 人工調整
```

### 補休

```text
compensatory_leave_credits
```

可能來源：

```text
1 = 加班產生
2 = 公司贈與
3 = 人工調整
```

而真正的公司贈與來源：

```text
company_leave_grants
```

會保留完整資料。

---

# 六、特別注意：贈與特休不能直接等同「法定特休」

這個很重要。

例如：

```text
法定特休
7 天

公司贈與特休
2 天
```

系統應該知道：

```text
法定來源：7 天
公司來源：2 天
```

不能最後只變成：

```text
特休 = 9 天
```

然後來源全部消失。

因為兩者可能有不同：

- 生效期間
- 到期日
- 遞延規則
- 到期處理
- 公司福利規則

所以一定要保留**來源批次**。

---

# 七、贈與補休也一樣

例如：

```text
加班產生補休
4 小時

公司贈與補休
8 小時
```

不能只變成：

```text
補休 = 12 小時
```

而應該是：

```text
Batch A
來源：加班
4 小時

Batch B
來源：公司贈與
8 小時
```

這樣我們原本的：

> **最早到期優先**

才能正確運作。

---

# 八、公司贈與不能直接修改額度

例如：

```text
公司贈與
8 小時
```

後來 HR 發現錯了。

不能：

```text
UPDATE granted_minutes = 4
```

而應該：

```text
原贈與 8h
↓
撤銷
↓
重新贈與 4h
```

這跟我們之前確定的：

> **核發後可以撤銷，再重新核發**

完全一致。

---

# 九、更新後的整體架構

現在會變成：

```text
                         company_leave_grants
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
              贈與特休                       贈與補休
                    │                           │
                    ▼                           ▼
          leave_entitlements       compensatory_leave_credits
                    │                           │
                    ▼                           ▼
             leave_balances          compensatory_leave_transactions
                    │
                    │
                    ▼
              leave_requests
                    │
                    ▼
          leave_request_details
                    │
                    ▼
          leave_request_allocations
```

而另一條補休來源：

```text
overtime
   │
   ▼
overtime_compensations
   │
   ▼
compensatory_leave_credits
```

所以現在就非常清楚：

```text
補休來源
├── 加班
├── 公司贈與
└── 人工調整

特休來源
├── 法定取得
├── 公司贈與
├── 遞延
└── 人工調整
```

---

## 十、我會順便修正上一版的一個設計

上一版我把：

```text
special_leave_entitlements
```

獨立出來。

現在加入「公司贈與特休」後，我反而建議**不要讓特休有兩套額度系統**。

統一成：

```text
leave_entitlements
```

然後：

```text
leave_type = 特休
source = 法定
```

或：

```text
leave_type = 特休
source = 公司贈與
```

都進同一套額度管理。

而 `special_leave_entitlements` 可以取消，**特休的特殊年資計算放到 `leave_type_rules` 處理**。

這樣整體會乾淨很多。

### 最終：

```text
leave_type_rules
       │
       ▼
leave_entitlements
       │
       ├── 法定特休
       ├── 公司贈與特休
       ├── 遞延特休
       └── 人工調整
```

這個版本我認為比前一版更合理，也符合我們一直在追求的：

> **來源清楚、額度分批、歷史可追蹤，而且不要為同一種假別建立兩套完全不同的帳務邏輯。**

---

## 2026/08/22 22:28:29

### 使用者

贈與的部分要分有薪贈與還是無薪贈與

### 助理

對，**這個一定要加入**。

而且我建議不要放在 `leave_types`，而是放在 **`company_leave_grants`**。

因為同一種「贈與假」可能有：

```text
公司贈與特休 → 有薪
公司贈與特休 → 無薪
公司贈與補休 → 有薪
公司贈與補休 → 無薪
```

所以「有薪／無薪」是**這一次贈與的屬性**，不是假別本身的屬性。

---

# `company_leave_grants` 修正版

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 公司贈與 ID |
| `employee_id` | `bigint` | 必填 | 被贈與的員工 |
| `leave_type_id` | `bigint` | 必填 | 贈與假別 |
| `grant_type_code` | `int` | 必填 | `1=有薪贈與`、`2=無薪贈與` |
| `granted_minutes` | `int` | 必填 | 贈與額度 |
| `effective_from` | `date` | 必填 | 生效日期 |
| `effective_to` | `date` | 選填 | 到期日期 |
| `reason` | `text` | 必填 | 贈與原因 |
| `status_code` | `int` | 必填 | `1=待核發`、`2=已核發`、`3=撤銷` |
| `granted_by` | `bigint` | 必填 | 核發者 |
| `granted_at` | `datetime` | 選填 | 核發時間 |
| `cancelled_by` | `bigint` | 選填 | 撤銷者 |
| `cancelled_at` | `datetime` | 選填 | 撤銷時間 |
| `cancel_reason` | `text` | 選填 | 撤銷原因 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

---

## 為什麼不用 `is_paid`

我會比較推薦：

```text
grant_type_code int
```

而不是：

```text
is_paid boolean
```

因為現在只有：

```text
1 = 有薪
2 = 無薪
```

但 HR 系統未來很可能會遇到第三種：

```text
3 = 依規則計算
```

例如某些公司福利假可能不是單純「全薪／無薪」，而是有特殊薪資計算方式。

所以用 `code` 比 boolean 有擴充性。

---

# 更重要的一點

這個「有薪／無薪」**不能只影響請假畫面**。

它最後要一路傳到薪資計算。

例如：

```text
公司贈與特休
+ 8 小時
有薪
```

員工請：

```text
2026/09/10
8 小時
```

薪資系統應該知道：

```text
leave_request
      ↓
leave_request_detail
      ↓
leave_request_allocation
      ↓
company_leave_grant
      ↓
grant_type_code = 有薪
      ↓
薪資：正常計薪
```

另一筆：

```text
公司贈與補休
+ 8 小時
無薪
```

請假後：

```text
薪資：扣除該 8 小時
```

---

# 但這裡出現一個重要問題

如果我們把：

```text
grant_type_code
```

放在 `company_leave_grants`，那麼當它轉成：

```text
leave_entitlements
```

時，**有薪／無薪資訊不能丟掉**。

所以我會再調整 `leave_entitlements`：

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 額度 ID |
| `employee_id` | `bigint` | 必填 | 員工 |
| `leave_type_id` | `bigint` | 必填 | 假別 |
| `source_type_code` | `int` | 必填 | `1=法定`、`2=公司贈與`、`3=遞延`、`4=人工調整` |
| `source_id` | `bigint` | 選填 | 原始來源 ID |
| `pay_type_code` | `int` | 必填 | `1=有薪`、`2=無薪` |
| `entitled_minutes` | `int` | 必填 | 額度 |
| `effective_from` | `date` | 必填 | 生效日 |
| `effective_to` | `date` | 選填 | 到期日 |
| `status_code` | `int` | 必填 | 額度狀態 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

### 為什麼這樣？

因為最後真正拿來請假的是：

```text
leave_entitlements
```

所以薪資系統不應該每次再回頭查：

```text
company_leave_grants
```

才能知道這 8 小時是不是有薪。

---

# 我會把概念定成這樣

```text
                公司贈與
                   │
                   ▼
        company_leave_grants
                   │
          ┌────────┴────────┐
          │                 │
       有薪贈與            無薪贈與
          │                 │
          └────────┬────────┘
                   ▼
          leave_entitlements
                   │
                   ▼
          leave_requests
                   │
                   ▼
             薪資結算
```

而且要特別注意：

**法定假別本身的 `is_paid` 仍然可以保留作為「預設值」，但實際請假時應以 entitlement / 該次額度的薪資屬性為準。**

這樣就能同時支援：

- 法定有薪假
- 法定無薪假
- 公司有薪贈與
- 公司無薪贈與
- 未來公司特殊薪資規則

我認為這個調整應該直接納入我們目前的請假 Schema 定案。

---

## 2026/08/22 22:29:37

### 使用者

思考一下有沒有問題

### 助理

有，而且我重新從「**實際 HR 使用 + 薪資結算 + 法規變動 + 歷史追蹤**」去檢查後，發現有幾個地方值得現在就修正。

我先不急著增加很多表，先把**設計上的坑抓出來**。

---

# 1. `pay_type_code` 放在 `leave_entitlements` 是對的，但要做 Snapshot

目前：

```text
leave_entitlements
    pay_type_code
```

這方向正確。

但是一旦這筆額度產生後，就**不能因為公司後來修改假別設定而跟著改變**。

例如：

```text
2026/01/01
公司贈與特休 2 天
有薪
```

之後 HR 把「公司贈與特休」預設改成：

```text
無薪
```

2026/01/01 那筆仍然必須是：

```text
有薪
```

所以：

> `leave_entitlements.pay_type_code` 是該批額度的歷史 Snapshot。

不能每次去 `leave_types` 查。

---

# 2. `leave_types.is_paid` 其實可能不需要

這是我重新思考後會修改的地方。

我們原本：

```text
leave_types
    is_paid
```

但現在有：

```text
leave_type_rules
    ↓
leave_entitlements
    ↓
pay_type_code
```

真正決定薪資的是「**這一批額度**」。

因此 `leave_types.is_paid` 很容易造成兩個來源：

```text
leave_types.is_paid
leave_entitlements.pay_type_code
```

最後到底誰是真的？

### 我建議：

**拿掉 `leave_types.is_paid`。**

改成：

```text
leave_type_rules
    pay_type_code
```

作為「規則預設值」

產生額度時：

```text
leave_type_rules.pay_type_code
            ↓
leave_entitlements.pay_type_code
```

後者成為不可變的 Snapshot。

這樣責任很清楚。

---

# 3. 公司贈與不能任意指定所有假別

目前：

```text
company_leave_grants
    leave_type_id
```

表面上沒問題。

但實際上可能有人誤操作：

```text
公司贈與
→ 公傷病假
```

這就不合理。

所以 `leave_types` 應該增加：

```text
grantable_by_company
boolean
```

例如：

| 假別 | 公司贈與 |
|---|---|
| 特休 | ✅ |
| 補休 | ✅ |
| 福利假 | ✅ |
| 事假 | ❌ |
| 公傷病假 | ❌ |
| 產假 | ❌ |
| 喪假 | ❌ |

這是**系統能力限制**，不是只靠 UI 防止。

---

# 4. 公司贈與「特休」和「補休」其實不完全相同

這是我認為比較重要的一個問題。

我們現在：

```text
company_leave_grants
```

可以：

```text
特休
補休
```

這很好。

但兩者後面的額度系統不同：

```text
公司贈與特休
→ leave_entitlements

公司贈與補休
→ compensatory_leave_credits
```

因此不能設計成：

```text
company_leave_grants
       ↓
統一直接產生 leave_entitlements
```

應該是：

```text
company_leave_grants
       │
       ├── 特休 ──→ leave_entitlements
       │
       └── 補休 ──→ compensatory_leave_credits
```

這點要保留。

---

# 5. 「有薪贈與補休」要小心定義

這是我們現在最值得釐清的地方。

例如公司贈與：

```text
補休 8 小時
有薪
```

這其實跟「加班換補休」的法律／薪資來源概念不完全相同。

所以系統上應該把：

```text
source_type_code
```

保留。

例如：

```text
補休來源

1 = 加班換休
2 = 公司贈與
3 = 人工調整
```

然後：

```text
pay_type_code
```

再表示：

```text
1 = 有薪
2 = 無薪
```

**來源與薪資性質是兩件不同的事。**

這個設計很好。

---

# 6. 請假時不能只知道「假別」，還要知道「實際扣哪一批額度」

這個我們之前已經有：

```text
leave_request_allocations
```

這張非常重要。

例如：

```text
特休 A
2026/01/01～06/30
有薪
剩 8h

特休 B
2026/07/01～12/31
無薪
剩 8h
```

員工請：

```text
8h
```

系統必須知道到底扣：

```text
A
```

還是：

```text
B
```

而不能只記：

```text
leave_type = 特休
8h
```

否則薪資會出問題。

---

# 7. `leave_request_details.pay_type_code` 要不要再存？

我重新想過後：

### 不建議直接再存一份

不要：

```text
leave_request_details.pay_type_code
```

因為會變成：

```text
leave_entitlements.pay_type_code
leave_request_details.pay_type_code
```

兩邊可能不一致。

真正的來源應該是：

```text
leave_request_details
        ↓
leave_request_allocations
        ↓
leave_entitlements
        ↓
pay_type_code
```

這樣資料只有一個權威來源。

---

# 8. 但薪資結算時要做 Snapshot

這跟我們之前的薪資設計是一致的。

例如：

```text
2026/08
員工請 8h
來源：
公司贈與特休
有薪
```

到了薪資結算：

```text
payroll
    ↓
取得 leave allocation
    ↓
取得 pay_type
    ↓
產生薪資計算結果
```

**結算後就不能回頭因為假別設定改了而重新解讀。**

所以薪資那邊最後還是會有自己的：

```text
payroll settlement snapshot
```

這跟我們之前「結算不能異動」的原則一致。

---

# 9. 公司贈與最好加入「核發批次／批次名稱」

這個是 UI 實務上很有用的。

例如公司一次送：

```text
2026 年生日福利
全體員工 +1 天特休
```

如果 500 個員工：

```text
company_leave_grants
500 筆
```

沒問題。

但是 HR 需要知道：

> 這 500 筆是同一次活動產生的。

所以我會增加：

```text
grant_batch_id
```

或另外建立：

```text
company_leave_grant_batches
```

但我現在**不建議急著開第二張表**。

第一階段可以先：

```text
company_leave_grants
    batch_no
```

例如：

```text
2026-BIRTHDAY-001
```

之後如果真的需要「批次操作／批次撤銷」，再獨立 Batch Table。

---

# 10. 公司贈與要支援「部分撤銷」嗎？

這個一定要先決定。

例如公司贈：

```text
10 小時
```

員工已經用了：

```text
6 小時
```

HR 發現原本應該只送：

```text
8 小時
```

這時候不能：

```text
整筆撤銷 10h
```

因為已經使用 6h。

所以公司贈與必須遵守：

```text
未使用額度
    ↓
可以撤銷

已使用額度
    ↓
不能直接抹掉
```

如果需要追回：

```text
建立額度調整交易
```

而不是修改原始贈與。

這點非常重要。

---

# 11. `company_leave_grants` 本身也應該是不可修改的業務紀錄

也就是：

```text
granted_minutes = 8h
```

核發後不能直接改成：

```text
granted_minutes = 4h
```

應該：

```text
原始贈與 8h
        ↓
保留
        ↓
撤銷／調整交易
        ↓
新的 4h
```

這跟我們之前補休的設計完全一致。

---

# 12. 贈與額度應該有「到期日」

這個已經有：

```text
effective_from
effective_to
```

但我會再強調：

**到期日不能只放在 `company_leave_grants`，產生額度時也要 Snapshot 到實際額度表。**

例如：

```text
company_leave_grants
effective_to = 2026/12/31
        ↓
leave_entitlements
effective_to = 2026/12/31
```

因為原始贈與規則未來可能被修改。

---

# 13. 公司贈與最好有「是否可遞延」

這是我認為現在還缺的一個規則。

例如：

```text
公司贈與特休 2 天
```

到期沒用完：

### A

```text
直接失效
```

### B

```text
可以遞延到明年
```

### C

```text
轉薪資
```

公司福利假未必跟法定特休一樣。

所以我建議 `company_leave_grants` 增加：

```text
expiry_rule_code
```

例如：

```text
1 = 到期失效
2 = 可遞延
3 = 到期轉薪資
4 = 依公司規則
```

但這個要和我們之前補休的：

> 到期未使用 → 轉薪資

**分開處理**。

因為公司贈與的特休不一定具有相同規則。

---

# 14. 最後一個：公司贈與「有薪／無薪」最好不要只有兩個狀態

目前：

```text
1 = 有薪
2 = 無薪
```

第一版可以。

但如果我們想讓這個 HR 系統長期使用，我會稍微改成：

```text
pay_rule_code
```

而不是：

```text
pay_type_code
```

例如：

```text
1 = 正常計薪
2 = 不給薪
3 = 半薪
4 = 依平均工資
5 = 依公司規則
```

因為「薪資怎麼處理」本質上是**薪資規則**。

---

# 我現在的建議

重新整理後，我會把這部分定成：

```text
leave_types
│
├── grantable_by_company
│
└── leave_type_rules
          │
          └── 預設 pay_rule
                    │
                    ▼
          company_leave_grants
                    │
          ┌─────────┴─────────┐
          │                   │
       特休贈與             補休贈與
          │                   │
          ▼                   ▼
leave_entitlements   compensatory_leave_credits
          │                   │
          └─────────┬─────────┘
                    ▼
             leave allocation
                    │
                    ▼
             payroll settlement
```

### 我認為現在需要確定的規則只剩 4 個：

1. **公司贈與的有薪／無薪，是核發時由核發者選擇，還是由公司預設規則決定？**
2. **公司贈與額度到期未使用：失效、遞延、還是轉薪資？是否可以每一筆贈與個別設定？**
3. **已使用部分的贈與額度，原則上不能撤銷，只能針對未使用部分撤銷／調整——我建議採這個。**
4. **公司贈與是否允許批次發放給多名員工？我建議要。**

這四個確認完，**公司贈與這一塊的 Schema 基本上就完整了**。

---

## 2026/08/22 22:33:37

### 使用者

1.同意2.同意3.同意4.同意5.同意6.同意7.同意8.同意9.同意加入10.不同意11.同意12.同意13.不同意14.只需要1 = 有薪 2 = 無薪

### 助理

了解。依你這次的決定，我把剛才 1～14 點重新整理，**不採用你否決的部分**，避免後面 Schema 越改越亂。

## 一、已確定採用

| # | 決定 | 最終規則 |
|---|---|---|
| 1 | ✅ 同意 | `pay_type_code` 放在實際額度，作為薪資屬性 Snapshot |
| 2 | ✅ 同意 | 不讓 `leave_types.is_paid` 與實際額度產生衝突，移除它 |
| 3 | ✅ 同意 | 假別增加 `grantable_by_company`，限制哪些假別可以公司贈與 |
| 4 | ✅ 同意 | 公司贈與特休、補休走各自的額度系統 |
| 5 | ✅ 同意 | 補休保留 `source_type_code`，區分加班產生／公司贈與等來源 |
| 6 | ✅ 同意 | 請假時薪資屬性從實際 allocation → entitlement 取得 |
| 7 | ✅ 同意 | 薪資結算時建立自己的 Snapshot |
| 8 | ✅ 同意 | 公司贈與增加批次概念 |
| 9 | ✅ **加入** | 公司贈與增加批次／批次編號 |
| 10 | ❌ 不同意 | **不另外設計「部分撤銷」機制** |
| 11 | ✅ 同意 | 公司贈與紀錄核發後不能直接修改 |
| 12 | ✅ 同意 | 贈與額度的到期日必須 Snapshot 到實際額度 |
| 13 | ❌ 不同意 | **不加入 `expiry_rule_code` 這個額外規則** |
| 14 | ⚠️ 修正 | `pay_type_code` 只需要 `1=有薪`、`2=無薪` |

---

# 二、特別修正第 10 點

你不同意我提出的「部分撤銷」。

所以我們不建立：

```text
partial_cancel
```

這類概念。

但我們仍然維持我們前面已經確定的原則：

> **原始核發紀錄不可直接修改。**

因此如果：

```text
公司贈與 10 小時
已使用 6 小時
```

原始資料仍然保持：

```text
10 小時
```

後續如果需要調整，就透過我們既有的**額度交易／調整機制**處理，而不是修改原始贈與資料。

---

# 三、第 13 點也正式取消

你不同意：

```text
expiry_rule_code
```

所以不做。

我們只保留：

```text
effective_from
effective_to
```

也就是：

> **這一批贈與什麼時候生效、什麼時候到期。**

至於到期後怎麼處理，依該額度所屬的業務規則處理，不在 `company_leave_grants` 再增加一套規則。

這樣 Schema 會乾淨很多。

---

# 四、`pay_type_code` 最終固定

就兩種：

```text
1 = 有薪
2 = 無薪
```

不做：

```text
3 = 半薪
4 = 平均工資
5 = 依公司規則
```

這個決定我認為很好。

因為「半薪、平均工資」這些其實應該由**法定假別的薪資計算規則**處理，而不是讓「公司贈與」這個功能無限制擴充。

---

# 五、目前公司贈與 Schema

因此目前可以正式定成：

### `company_leave_grants`

| 欄位 | 型態 | 必要性 | 說明 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 公司贈與 ID |
| `batch_no` | `varchar(30)` | 必填 | 贈與批次編號 |
| `employee_id` | `bigint` | 必填 | 被贈與員工 |
| `leave_type_id` | `bigint` | 必填 | 贈與假別 |
| `pay_type_code` | `int` | 必填 | `1=有薪`、`2=無薪` |
| `granted_minutes` | `int` | 必填 | 贈與額度 |
| `effective_from` | `date` | 必填 | 生效日期 |
| `effective_to` | `date` | 選填 | 到期日期 |
| `reason` | `text` | 必填 | 贈與原因 |
| `status_code` | `int` | 必填 | `1=待核發`、`2=已核發`、`3=撤銷` |
| `granted_by` | `bigint` | 必填 | 核發者 |
| `granted_at` | `datetime` | 選填 | 核發時間 |
| `cancelled_by` | `bigint` | 選填 | 撤銷者 |
| `cancelled_at` | `datetime` | 選填 | 撤銷時間 |
| `cancel_reason` | `text` | 選填 | 撤銷原因 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 更新時間 |

---

# 六、批次贈與

例如 HR 要做：

> 「2026 年生日福利，全公司每人贈與 8 小時有薪特休」

我們可以產生：

```text
batch_no = 2026-BIRTHDAY-001
```

然後：

```text
員工 A → 8h
員工 B → 8h
員工 C → 8h
員工 D → 8h
...
```

每個員工仍然是一筆 `company_leave_grants`。

這樣：

- 可以查某次活動送了誰
- 可以查某員工收到哪些公司贈與
- 可以統一查批次
- 不會把多個員工塞成一筆資料

---

# 七、最終資料流

目前我們的設計會變成：

```text
                    公司贈與
                       │
                       ▼
              company_leave_grants
                       │
             ┌─────────┴─────────┐
             │                   │
          特休贈與             補休贈與
             │                   │
             ▼                   ▼
   leave_entitlements   compensatory_leave_credits
             │                   │
             ▼                   ▼
      leave_balances      補休交易帳本
             │
             ▼
       leave_requests
             │
             ▼
   leave_request_allocations
             │
             ▼
      Payroll Settlement
```

而且整個架構現在有一個很重要的原則：

> **「假別」是什麼、 「額度從哪裡來」、「這批額度有沒有薪」、「這批額度何時到期」全部分開。**

這樣後面我們繼續設計「請假制度」時，才不會因為特休、補休、公司贈與、法定假別混在一起而變得很難維護。

---

## 2026/08/22 22:34:04

### 使用者

在思考一下，還有問題嗎

### 助理

有。我再從「**資料一致性、實際 HR 操作、薪資、法規變更、撤銷與歷史追蹤**」重新走一次，目前還有幾個值得現在就決定的問題。

我認為其中 **3 個是必須處理的**。

---

## 1. 公司贈與的 `batch_no` 不應該直接放在每筆贈與資料裡

目前：

```text
company_leave_grants
    batch_no
    employee_id
    ...
```

可以運作，但有一個問題：

假設 HR 建立：

> 2026 生日福利

批次本身其實還有：

- 批次名稱
- 贈與原因
- 核發人
- 建立時間
- 預計贈與人數
- 批次狀態

如果全部塞在每一筆 `company_leave_grants`：

```text
員工 A → 2026-BIRTHDAY-001
員工 B → 2026-BIRTHDAY-001
員工 C → 2026-BIRTHDAY-001
```

資料會重複。

### 我建議正式拆成：

```text
company_leave_grant_batches
        │
        └── company_leave_grants
```

### `company_leave_grant_batches`

```text
id
batch_no
name
reason
status_code
created_by
created_at
updated_at
```

### `company_leave_grants`

```text
id
batch_id
employee_id
leave_type_id
pay_type_code
granted_minutes
effective_from
effective_to
status_code
granted_by
granted_at
...
```

這樣比較乾淨。

---

# 2. 公司贈與「撤銷」的定義還需要鎖定

我們已經確定：

> 核發後不能直接修改。

這很好。

但現在 `company_leave_grants.status_code = 3` 是「撤銷」。

問題是：

### 如果這筆贈與已經被員工使用呢？

例如：

```text
公司贈與特休 8h
↓
員工使用 4h
↓
HR 想撤銷
```

我們不能把：

```text
8h → 0h
```

因為員工已經合法使用 4h。

所以我建議定義：

> **公司贈與撤銷，只能撤銷尚未被使用的額度；已使用部分不得被原始撤銷動作抹除。**

如果已經使用：

```text
8h 贈與
4h 已使用
4h 尚未使用
```

撤銷時：

```text
原贈與：8h
保留歷史
未使用：4h → 撤銷
已使用：4h → 保留
```

這不叫「部分撤銷」，而是：

> **撤銷尚未使用的剩餘額度。**

我建議採這個定義。

---

# 3. `granted_minutes` 要不要允許 0？

我建議：

**不允許 0 或負數。**

```text
granted_minutes > 0
```

因為：

```text
company_leave_grants
```

本身代表「贈與」。

如果要扣除：

```text
不能：
granted_minutes = -8
```

而應該走：

```text
leave_balance_transactions
```

或：

```text
compensatory_leave_transactions
```

的調整機制。

這樣：

> **原始紀錄永遠代表原始事實。**

---

# 4. 公司贈與是否需要「指定贈與人員」

這個我認為一定要。

批次建立後可能：

```text
全公司
某部門
某職位
指定員工
```

但我**不建議把部門、職位條件直接塞進 `company_leave_grant_batches`**。

因為你現在的人事設計裡：

> 員工可以有多個職務，但部門歷史會變。

如果今天：

```text
行銷部
全員贈與 8h
```

明天員工轉部門，再回頭看歷史，很容易產生歧義。

所以最終贈與結果一定要落成：

```text
employee_id
```

批次只是「這次活動怎麼產生」，實際贈與對象還是員工。

---

# 5. 批次建立後，如果員工離職怎麼辦？

例如：

```text
8/1 建立贈與批次
8/5 核發
8/10 員工離職
```

這筆額度不能因為員工離職就被刪除。

應該：

```text
employee
   ↓
company_leave_grant
   ↓
leave_entitlement
```

完整保留。

至於離職結算時是否要處理未使用的公司贈與，則交由：

> **離職結算規則**

處理。

不要在 `company_leave_grants` 裡硬塞。

---

# 6. 贈與特休是否真的叫「特休」？

這個是業務上的小陷阱。

如果公司說：

> 「額外送 3 天特休」

在系統裡最好仍然是：

```text
leave_type = 公司福利特休
```

而不是直接塞進：

```text
法定特別休假
```

因為未來可能需要區分：

```text
法定特休
公司贈與特休
```

兩者來源不同、規則可能不同。

### 我反而建議建立：

```text
leave_type
    LEGAL_ANNUAL
    COMPANY_ANNUAL
    COMPENSATORY
```

而不是：

```text
ANNUAL
source = legal/company
```

**這一點我目前比較傾向前者。**

因為 UI、報表、薪資、離職結算都會比較清楚。

---

# 7. 但補休不需要這樣做

補休我們已經有：

```text
compensatory_leave_credits
```

所以：

```text
加班產生補休
公司贈與補休
```

可以共用：

```text
leave_type = COMPENSATORY
source_type
```

不需要再做：

```text
COMPENSATORY_OT
COMPENSATORY_COMPANY
```

因為補休本身的帳本已經有來源。

---

# 8. `pay_type_code` 的權威來源要固定

這點現在我建議直接定死：

```text
company_leave_grants.pay_type_code
          ↓
leave_entitlements.pay_type_code
          ↓
leave_request_allocations
          ↓
payroll
```

其中：

### `company_leave_grants`

是：

> 公司核發時決定。

### `leave_entitlements`

是：

> 核發後的 Snapshot。

### `leave_requests`

**不存。**

### `payroll`

結算時再建立自己的 Snapshot。

這樣不會出現：

```text
請假單說有薪
額度說無薪
薪資又說有薪
```

---

# 9. 我發現一個更重要的問題：公司贈與可以「直接核發」還是一定要申請？

我建議：

```text
公司贈與
不是員工申請
```

而是：

```text
HR / 有權限人員
      ↓
建立贈與批次
      ↓
核發
      ↓
直接產生額度
```

也就是：

```text
company_leave_grant_batches
        ↓
company_leave_grants
        ↓
leave_entitlements
```

不應該走：

```text
leave_requests
```

因為這不是「員工請假」。

---

# 10. 最後一個我認為要先定義：公司贈與能不能「立即生效」

我建議可以。

例如：

```text
建立贈與
生效日 = 今天
核發
```

就立即進入：

```text
leave_entitlements
```

如果：

```text
生效日 = 下個月 1 號
```

則：

```text
公司贈與已核發
但額度尚未可使用
```

所以：

```text
granted_at
```

和：

```text
effective_from
```

一定要分開。

這一點我們目前已經有了，是正確的。

---

# 我現在的結論

如果要我替這套設計做一次 Code Review，我會說：

### 必須修正

**① 增加 `company_leave_grant_batches`**

```text
批次
 ↓
多筆員工贈與
```

**② 明確定義撤銷**

> 只能撤銷尚未使用的額度；已使用部分保留。

**③ `granted_minutes > 0`**

不能拿贈與紀錄當調整紀錄。

---

### 已經很穩定的部分

```text
公司贈與
├── 有薪 / 無薪
├── 特休 / 補休
├── 生效日
├── 到期日
├── 核發者
├── 撤銷
├── 批次
└── 完整歷史
```

以及：

```text
法定額度
公司贈與額度
遞延額度
人工調整額度
```

都可以進入額度系統，但**來源永遠保留**。

我認為把上面 3 個必須修正的地方處理掉後，這部分就可以進入「**請假 Schema 最終版**」，不用再一直增加欄位。

---

## 2026/08/22 22:37:55

### 使用者

1.同意2.同意3.同意4.同意5.同意6.是特休，福利假歸福利假7.同意8.同意9.直接核發10.能指定期使跟結束日

### 助理

好，這樣就完整很多了。尤其第 6 點我會正式修正成：

> **特休就是特休，福利假就是福利假，兩者是不同假別。**

另外第 10 點也確定：

> **公司贈與可以指定「開始日」與「結束日」**，不一定是核發當天立即生效。

所以最終會是：

```text
company_leave_grant_batches
        │
        ▼
company_leave_grants
        │
        ├── employee_id
        ├── leave_type_id
        ├── pay_type_code       1=有薪 / 2=無薪
        ├── granted_minutes
        ├── effective_from     ← 開始日
        ├── effective_to       ← 結束日
        ├── granted_by
        └── granted_at         ← 實際核發時間
```

例如：

> 公司 8/22 核發 2026 年生日特休 1 天，但規定 9/1～12/31 才能使用。

資料就是：

```text
granted_at       = 2026-08-22
effective_from   = 2026-09-01
effective_to     = 2026-12-31
```

因此**核發時間 ≠ 可使用期間**，這個設計可以處理。

目前公司贈與這一塊，我認為可以先定案，不需要再繼續往裡面塞欄位了。

---

## 2026/08/22 22:38:16

### 使用者

還有問題嗎

### 助理

有，我再做一次最後的「反向測試」。目前沒有需要推翻架構的大問題，但還有 **5 個規則最好現在定下來**，否則做到請假 UI / 額度計算時會遇到歧義。

### 1. 生效日與結束日是否允許只填其中一個？

我建議：

- `effective_from`：**必填**
- `effective_to`：**可空**
- 如果有結束日，必須 `effective_to >= effective_from`

例如：

```text
2026/09/01 ～ 2026/12/31
```

或：

```text
2026/09/01 ～ 無期限
```

---

### 2. 已經開始使用後，可以修改「結束日」嗎？

例如：

```text
贈與特休 16 小時
有效期：
2026/01/01 ～ 2026/12/31

員工已使用 8 小時
```

HR 發現應該改成：

```text
2026/01/01 ～ 2026/10/31
```

我建議：

> **不能直接修改原始贈與紀錄。**

如果要調整，走「額度調整／撤銷」機制，保留完整歷史。

這樣才能符合我們前面確定的：

> 核發後不能直接修改。

---

### 3. 生效前可以撤銷嗎？

例如：

```text
8/22 核發
9/1 開始生效
```

8/25 HR 發現送錯。

這時還沒開始使用，我建議：

> **可以直接撤銷。**

因為沒有任何已使用額度，不會破壞後面的請假紀錄。

---

### 4. 贈與額度過期後，要不要自動失效？

你已經決定有：

```text
effective_to
```

我建議系統：

> **到期後自動判定為不可使用，但不要刪除額度。**

例如：

```text
贈與 8h
有效期到 12/31

2027/01/01
↓
剩餘 3h
↓
不可再申請
↓
歷史仍保留 3h
```

不要真的把：

```text
8h → 0h
```

因為那會破壞歷史。

---

### 5. 批次贈與後，某個員工能不能被排除？

例如：

> 全公司贈與 8 小時特休

但其中：

```text
員工 A
員工 B
員工 C
```

其中 B 不符合資格。

我建議批次建立時可以：

```text
全體符合條件
        ↓
產生員工贈與明細
        ↓
排除特定員工
```

而不是先產生再刪除。

---

## 我目前認為最重要的是這個

我們現在已經有三個不同概念：

```text
核發
↓
生效
↓
使用
```

例如：

```text
8/22      公司核發
   ↓
9/1       額度開始可以使用
   ↓
9/10      員工請假 8 小時
   ↓
12/31     額度到期
```

這四個時間點不能混在一起。

所以 `company_leave_grants` 最終至少要維持：

```text
granted_at
effective_from
effective_to
```

而真正「用了多少」則由額度／allocation 系統記錄。

---

### 如果以上 5 點都採我的建議

那我會認為：

**「公司贈與」這個子系統可以正式定案，不需要再繼續加功能。**

接下來就可以把它整合回我們正在設計的**完整請假 Schema**，包括：

```text
假別
↓
法定假額度
公司贈與
補休
請假申請
審核
額度扣除
撤銷
到期
薪資 Snapshot
```

這樣才是下一階段真正需要處理的整體。

---

## 2026/08/22 22:39:26

### 使用者

1.都要填
2.不行
3.可以
4.到期後自動判定為不可使用，但不要刪除額度。  
5.可以

### 助理

好，這 5 點確認後，**公司贈與制度可以正式定案**。我把最終規則整理成一版，後面 Schema 就以這版為準。

## 公司贈與最終規則

### 1. 生效開始日、結束日

兩個都必填：

```text
effective_from
effective_to
```

而且：

```text
effective_to >= effective_from
```

不允許無期限贈與。

---

### 2. 核發後不能修改期間

一旦核發：

```text
effective_from
effective_to
```

都不能直接修改。

例如：

```text
原本：
2026/09/01 ～ 2026/12/31
```

不能直接改成：

```text
2026/09/01 ～ 2027/01/31
```

如果真的需要調整，要走既有的**撤銷／額度調整機制**，保留原始歷史。

---

### 3. 尚未生效可以撤銷

例如：

```text
核發：2026/08/22
生效：2026/09/01
```

在 9/1 前：

```text
可以撤銷
```

而且因為尚未使用，所以可以完整撤銷這筆贈與。

---

### 4. 到期自動不可使用

到：

```text
effective_to
```

之後：

```text
額度仍然存在
        ↓
不可再使用
        ↓
歷史資料保留
```

**不刪除、不把原始額度改成 0。**

這點非常重要，因為之後報表仍然可以知道：

> 公司當初贈與多少、使用多少、剩多少、什麼時候到期。

---

### 5. 批次贈與可以排除員工

例如：

```text
2026 生日福利
全公司贈與 8 小時有薪特休
```

可以指定：

```text
全公司
```

再排除：

```text
員工 A
員工 B
```

最終仍然產生**每個員工自己的 `company_leave_grants` 紀錄**。

---

# 最終資料模型

```text
company_leave_grant_batches
        │
        │ 1:N
        ▼
company_leave_grants
        │
        ├── employee_id
        ├── leave_type_id
        ├── pay_type_code
        ├── granted_minutes
        ├── effective_from
        ├── effective_to
        ├── granted_by
        └── granted_at
                │
                ▼
        實際額度系統
                │
        ┌───────┴────────┐
        ▼                ▼
      特休             補休
        │                │
        ▼                ▼
leave_entitlements   compensatory_leave_credits
```

### 特別確認

我們目前已經定死：

```text
pay_type_code
1 = 有薪
2 = 無薪
```

假別：

```text
特休 = 特休
福利假 = 福利假
補休 = 補休
```

**不把福利假併進特休。**

公司贈與也不是員工申請，而是：

```text
有權限的人
    ↓
建立贈與批次
    ↓
指定員工
    ↓
直接核發
    ↓
產生額度
```

這一塊現在我會視為**已定案**。接下來整合「請假申請 Schema」時，就直接引用這些規則，不再重新設計公司贈與。

