# 2026-08-22 對話逐字稿 — Part 004

---

## 2026/08/22 20:54:14

### 使用者

部門的歷史追蹤很重要?

### 助理

對，**如果是完整 HR SaaS，我會認為部門歷史追蹤很重要**，但原因不是單純「方便查看」，而是它會直接影響薪資、權限、報表、組織管理等後續功能。

例如員工：

```text
2024/01/01
研發部 → 研發一處

2025/07/01
研發一處 → 研發二處

2026/03/01
研發二處 → 產品部
```

如果 `employees` 只有：

```text
department_id = 產品部
```

那你只能知道**現在在哪裡**，完全不知道以前在哪裡。

---

### HR 實務上會用到

例如主管問：

> 「2025 年研發一處有多少人？」

如果沒有歷史，就很難正確回答。

或者：

> 「這個員工 2025 年 8 月的薪資成本算在哪個部門？」

這就更需要知道**當時的部門**。

---

所以我會建議：

```text
employees
    │
    └── employee_employments
            │
            ├── 任職期間
            ├── department_id
            └── job_title_id
```

例如：

| 任職紀錄 | 到職 | 離職 | 部門 | 職稱 |
|---|---|---|---|---|
| E001 | 2024/01/01 | 2025/06/30 | 研發一處 | 工程師 |
| E002 | 2025/07/01 | 2026/02/28 | 研發二處 | 資深工程師 |
| E003 | 2026/03/01 | NULL | 產品部 | 產品經理 |

---

不過有一個更細的問題：

**部門異動本身也應該留下歷史。**

因為如果只是把 `department_id` 放在 `employee_employments`：

```text
employee_employments
department_id = 研發部
```

員工轉部門時，如果修改這筆資料，歷史還是會消失。

所以比較完整的設計其實會是：

```text
employees
    │
    └── employee_employments
            │
            ├── employee_department_histories
            │
            └── employee_job_title_histories
```

但**現在先不用急著建立這兩張**。

我建議我們下一步先把「**任職紀錄到底需要哪些欄位**」定義清楚，再決定部門、職稱歷史要不要獨立。這樣 Schema 會比較穩。

---

## 2026/08/22 20:54:54

### 使用者

可以那一你的建議調整

### 助理

可以，我建議現在就把架構調整成「**人員主檔 → 任職紀錄 → 任職異動歷史**」的方向。

這樣能同時解決：

- 離職後再回任
- 部門異動
- 職稱異動
- 未來薪資異動
- 查詢某個時間點員工的任職狀態

而且我們現在仍然**只討論 Schema**。

---

# 一、`employees`

**說明：**  
儲存「這個人」本身的基本資料。只保存員工個人資料，不保存任職期間、部門、職稱等具有任職歷史性的資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工系統唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID，關聯 `companies.id` |
| `employee_code` | `string` | 員工在公司內的識別編號 |
| `name` | `string` | 員工姓名 |
| `gender` | `string` | 性別代碼，不使用 DB ENUM |
| `identity_number_encrypted` | `binary` | 身分證統一編號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證統一編號 Hash，用於比對 |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `phone_encrypted` | `binary` | 電話，加密儲存 |
| `email_encrypted` | `binary` | Email，加密儲存 |
| `address_encrypted` | `binary` | 居住地址，加密儲存 |
| `status` | `string` | 員工資料狀態，不使用 DB ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

---

# 二、`employee_employments`

**說明：**  
儲存員工每一次與公司的任職關係。一名員工可以有多段任職期間，因此員工離職後再次回任時，建立新的任職紀錄。

例如：

```text
員工 A

第一次任職
2020/01/01 → 2023/06/30

第二次任職
2025/03/01 → 目前
```

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 任職紀錄唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 員工 ID，關聯 `employees.id` |
| `hire_date` | `date` | 該次任職的到職日期 |
| `leave_date` | `date` | 該次任職的離職日期，可為 NULL |
| `status` | `string` | 該次任職狀態，不使用 DB ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

### 特殊規則

同一員工可以有多筆：

```text
employee_employments
```

但同一時間原則上只能有一個有效任職期間。

---

# 三、`employee_department_histories`

**說明：**  
記錄員工在任職期間所屬部門的歷史異動。員工轉換部門時建立新紀錄，不修改舊資料，以保留完整的部門異動歷史。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 部門異動紀錄唯一識別碼，主鍵 |
| `employment_id` | `uuid` | 任職紀錄 ID，關聯 `employee_employments.id` |
| `department_id` | `uuid` | 部門 ID，關聯 `departments.id` |
| `start_date` | `date` | 開始隸屬該部門日期 |
| `end_date` | `date` | 結束隸屬該部門日期，可為 NULL |
| `reason` | `string` | 部門異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |

例如：

```text
2024/01/01 → 2025/06/30
研發一處

2025/07/01 → 2026/02/28
研發二處

2026/03/01 → NULL
產品部
```

---

# 四、`employee_job_title_histories`

**說明：**  
記錄員工在任職期間的職稱歷史。升遷、降職或職稱調整時建立新的紀錄，不修改原有資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職稱異動紀錄唯一識別碼，主鍵 |
| `employment_id` | `uuid` | 任職紀錄 ID，關聯 `employee_employments.id` |
| `job_title_id` | `uuid` | 職稱 ID，關聯 `job_titles.id` |
| `start_date` | `date` | 開始使用該職稱日期 |
| `end_date` | `date` | 結束使用該職稱日期，可為 NULL |
| `reason` | `string` | 職稱異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |

例如：

```text
2024/01/01 → 2025/06/30
工程師

2025/07/01 → 2026/02/28
資深工程師

2026/03/01 → NULL
產品經理
```

---

# 五、`job_titles`

前面討論的職稱資料表保留。

**說明：**  
儲存系統預設及公司自訂的職稱。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職稱唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID，可為 NULL；NULL 表示系統預設 |
| `code` | `string` | 職稱代碼 |
| `name` | `string` | 職稱名稱 |
| `description` | `string` | 職稱說明 |
| `is_system` | `boolean` | 是否為系統預設職稱 |
| `status` | `string` | 職稱狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

---

# 六、薪資歷史也掛到任職紀錄

這裡我會再做一個調整。

之前：

```text
employee_salary_histories
    ↓
employee_id
```

現在建議改成：

```text
employee_salary_histories
    ↓
employment_id
```

因為員工可能：

```text
第一次任職
35,000 → 40,000
```

離職後：

```text
第二次任職
45,000 → 50,000
```

這兩段任職的薪資歷史應該分開。

---

# 七、`employee_salary_histories`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資歷史紀錄唯一識別碼，主鍵 |
| `employment_id` | `uuid` | 任職紀錄 ID，關聯 `employee_employments.id` |
| `salary` | `decimal` | 該期間基本薪資 |
| `effective_date` | `date` | 薪資開始生效日期 |
| `end_date` | `date` | 薪資結束日期，可為 NULL |
| `reason` | `string` | 薪資異動原因 |
| `description` | `string` | 薪資異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 八、扶養資料

`employee_dependents` 維持原本設計，仍然直接掛在 `employees`：

```text
employees
   │
   └── employee_dependents
```

因為扶養人是「這個人的資料」，不屬於某一次任職。

例如：

```text
員工
第一次任職
離職
第二次任職

扶養子女
仍然是同一個人的扶養資料
```

所以不應該綁 `employment_id`。

---

# 最後整體關係

```text
companies
    │
    └── employees
          │
          ├── employee_dependents
          │
          └── employee_employments
                  │
                  ├── employee_department_histories
                  │
                  ├── employee_job_title_histories
                  │
                  └── employee_salary_histories
```

另外：

```text
departments
      ↑
      │
employee_department_histories

job_titles
      ↑
      │
employee_job_title_histories
```

這個版本我認為比前面的結構完整很多。

**核心概念變成：**

> `employees` = 人  
> `employee_employments` = 這個人某一次任職  
> `*_histories` = 這次任職期間發生過什麼變化

這樣之後做**打卡、薪資計算、請假、報表、部門成本、員工異動紀錄**時，都比較不容易遇到歷史資料被覆蓋的問題。

---

## 2026/08/22 20:56:52

### 使用者

思考一下，這設計可能的問題

### 助理

有，我重新站在「真的要拿來做 HR SaaS」的角度檢查一次。現在這版已經比一開始合理很多，但我看到幾個**需要現在就決定的問題**。

## 1. 最大問題：`employees.company_id` 的意義

現在是：

```text
employees
├── id
├── company_id
└── ...
```

這代表：

> 一個 Employee 永遠只屬於一家公司。

但我們前面其實已經有一個很重要的 SaaS 特性：

> **同一個人可能跟不同公司有關係。**

例如某個人：

```text
A 公司 → 員工
B 公司 → 員工
```

如果 `employees` 本身綁 `company_id`，就會變成：

```text
employees
E001 → A公司
E002 → B公司
```

也就是**同一個人會有兩筆 employee**。

### 我認為這其實不一定是錯

反而對 HR SaaS 而言，我會傾向：

```text
employees = 某公司的人事資料
```

而不是：

```text
employees = 全世界這個人的唯一資料
```

原因是：

- A 公司不應該知道他在 B 公司是不是員工
- A 公司不應該看到 B 公司的資料
- SaaS Tenant 隔離比較乾淨
- 同一個人不同公司可以有不同員工編號、Email、地址、職稱

所以：

**我建議暫時保留 `employees.company_id`。**

不要現在為了「同一個人」再拆一張 `persons`，那會讓 HR 系統複雜非常多。

---

# 2. `employee_employments` 還缺一個重要概念：任職類型

現在只有：

```text
hire_date
leave_date
status
```

但 HR 通常會有：

```text
正職
兼職
約聘
派遣
工讀
臨時
顧問
實習
```

所以我建議增加：

```text
employment_type_code : integer
```

例如：

```text
1 = 正職
2 = 兼職
3 = 約聘
4 = 派遣
5 = 工讀
6 = 臨時
7 = 顧問
8 = 實習
```

同樣：

**不用 ENUM、不需要另外開資料表。**

直接寫在欄位 COMMENT。

---

# 3. `employee_employments.status` 可能跟 `employees.status` 重複

現在有：

```text
employees.status
employee_employments.status
```

這裡容易出問題。

例如：

```text
employees.status = active
employee_employments.status = terminated
```

到底誰是真的？

所以這兩個狀態必須分清楚。

### `employees.status`

代表：

> 這筆員工主資料目前是否有效

例如：

```text
1 = 啟用
2 = 停用
```

### `employee_employments.status`

代表：

> 這一次任職關係的狀態

例如：

```text
1 = 任職中
2 = 已離職
3 = 留職停薪
```

這樣才不會混在一起。

---

# 4. `job_title_histories` + `department_histories` 會有同步問題

現在：

```text
employee_employments
    │
    ├── employee_department_histories
    │
    └── employee_job_title_histories
```

假設：

```text
2026/01/01
研發部 / 工程師

2026/03/01
研發二處 / 資深工程師
```

兩張表都會有：

```text
2026/03/01
```

這沒有問題。

但如果：

```text
2026/03/01
部門變更

2026/04/01
職稱變更
```

兩張表的時間軸就各自獨立。

這其實是**合理的**，因為：

> 部門和職稱本來就是兩種不同的異動。

所以我**不建議現在合併成一張 `employee_assignment_histories`**。

分開反而比較容易維護。

---

# 5. 但是「職稱」可能還不等於「職位」

這個要特別注意。

例如：

```text
職稱：工程師
職位：後端工程師
```

或者：

```text
職稱：經理
職位：研發部經理
```

甚至：

```text
職稱：專員
職務：人資專員
```

所以目前我們先用：

```text
job_title
```

沒有問題。

但未來如果要做到比較完整的 HR：

```text
職稱
職務
職等
職級
職位
```

這幾個概念要分開。

**現在先不要擴充，避免過度設計。**

---

# 6. 薪資目前的 `salary` 太簡單

這是我認為下一個比較重要的問題。

現在：

```text
employee_salary_histories
└── salary
```

但真正計薪可能會有：

```text
基本薪資
職務加給
主管加給
交通津貼
伙食津貼
全勤
績效獎金
加班費
```

如果全部塞：

```text
salary = 50000
```

未來會不夠。

但是！

**我現在不建議馬上把所有薪資欄位塞進來。**

因為我們還沒討論「薪資制度」。

比較好的做法是先把：

```text
employee_salary_histories
```

定義成「**基本薪資歷史**」。

之後計薪模組再設計：

```text
薪資項目
薪資組成
津貼
獎金
扣款
```

這樣比較乾淨。

---

# 7. 扶養資料其實還有一個問題：年度

目前：

```text
effective_date
end_date
```

可以知道扶養關係何時開始、結束。

但**報稅是有年度概念的**。

例如：

```text
2025 年
扶養小孩

2026 年
小孩開始工作，不再扶養
```

未來做報稅時，你很可能需要知道：

> 「2025 年申報時，這個人是不是扶養人？」

所以將來可能需要：

```text
tax_year
```

或者建立「年度申報資料」。

但我現在**不建議直接加 `tax_year`**。

因為我們目前只是 HR 基礎資料，等真的討論「扣繳 / 報稅」時再決定比較好。

---

# 8. 身分證加密 + Hash 是合理的，但要注意用途

目前：

```text
identity_number_encrypted
identity_number_hash
```

這個設計我認同。

因為：

```text
encrypted
```

用於：

> 需要顯示身分證時解密

而：

```text
hash
```

用於：

> 查詢 / 比對是否已存在

例如：

```text
使用者輸入 A123456789

↓ Hash

identity_number_hash

↓ 查詢

找到員工
```

這比直接對加密欄位做搜尋合理。

---

# 9. `birthday_encrypted` 也有一個實務問題

如果生日完全加密：

```text
birthday_encrypted
```

那你不能很方便地做：

```text
今年滿 18 歲？
今年滿 65 歲？
生日月份？
年資 / 年齡相關報表？
```

不過這不代表不能加密。

而是未來如果真的有這類需求，可能會需要：

```text
birthday_encrypted
birthday_date / birthday_year
```

或者使用其他衍生資料。

**目前我建議先保持加密，不急著增加明文欄位。**

---

# 10. `employee_code` 的唯一性要明確

應該是：

```text
company_id + employee_code
```

唯一。

而不是：

```text
employee_code
```

全系統唯一。

因為：

```text
A公司
EMP001

B公司
EMP001
```

完全合理。

---

# 我目前會把整體架構定成這樣

```text
companies
    │
    └── employees
          │
          ├── employee_dependents
          │
          └── employee_employments
                  │
                  ├── employee_department_histories
                  │
                  ├── employee_job_title_histories
                  │
                  └── employee_salary_histories

job_titles ───────────────┐
                           │
departments ──────────────┤
                           │
                           ▼
                    各種 History
```

### 我認為現在最重要的設計原則是：

**Employee 是「人事主檔」**

```text
employees
```

**Employment 是「某一次任職」**

```text
employee_employments
```

**History 是「這次任職期間的變化」**

```text
department history
job title history
salary history
```

這個方向我會保留。

另外，我會建議**下一步先把 `employee_employments` 完整定下來**，因為它現在會成為整個員工 HR 資料的核心；像「任職類型、試用期、工號、到職、離職、離職原因、工作地點」等，都應該先在這一層想清楚。

---

## 2026/08/22 21:06:38

### 使用者

同意你說的，那薪資我們另外計畫，不要放入這一次，你把你的人事總結一下

### 助理

可以。薪資這次先**完全從人事基本資料範圍拿掉**，之後另外規劃「薪資/計薪模組」。

目前我們討論到的人事部分，我建議正式整理成下面這個版本。

# 人事資料 Schema 總結

整體核心概念：

```text id="9x8g2a"
公司
 │
 └── 員工
      │
      ├── 扶養親屬
      │
      └── 任職紀錄
            │
            ├── 部門歷史
            └── 職稱歷史
```

---

## 1. `employees`

**用途：員工主檔**

代表「這個人」，只放個人基本資料，不放到職、離職、部門、職稱等歷史性資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID |
| `employee_code` | `string` | 公司內員工編號 |
| `name` | `string` | 員工姓名 |
| `gender` | `string` | 性別代碼，不使用 ENUM |
| `identity_number_encrypted` | `binary` | 身分證字號，加密 |
| `identity_number_hash` | `binary` | 身分證字號 Hash，用於查詢比對 |
| `birthday_encrypted` | `binary` | 出生年月日，加密 |
| `phone_encrypted` | `binary` | 電話，加密 |
| `email_encrypted` | `binary` | Email，加密 |
| `address_encrypted` | `binary` | 居住地址，加密 |
| `status` | `string` | 員工主資料狀態，不使用 ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

### 特殊規則

```text id="4i0t6h"
company_id + employee_code
```

應具有唯一性。

---

# 2. `employee_employments`

**用途：員工任職紀錄**

這是整個人事模組的核心。

一個員工可以有多次任職：

```text id="kq0j0w"
員工 A

第一次任職
2020/01/01 → 2023/06/30

第二次任職
2025/03/01 → NULL
```

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 任職紀錄唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 員工 ID |
| `employment_type_code` | `integer` | 任職類型代碼 |
| `hire_date` | `date` | 該次任職到職日期 |
| `leave_date` | `date` | 該次任職離職日期，可 NULL |
| `status` | `string` | 該次任職狀態，不使用 ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

### `employment_type_code`

直接寫在欄位註釋：

```text id="o5p0h3"
1 = 正職
2 = 兼職
3 = 約聘
4 = 派遣
5 = 工讀
6 = 臨時
7 = 顧問
8 = 實習
```

之後如果發現實務上需要增加，只增加代碼，不改 Schema。

---

# 3. `departments`

前面已經確定的部門 Schema。

**用途：公司部門樹狀結構**

```text id="t8qz2p"
總經理室
└── 研發部
    ├── 研發一處
    └── 研發二處
```

核心欄位：

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 部門唯一識別碼 |
| `company_id` | `uuid` | 所屬公司 |
| `parent_id` | `uuid` | 上層部門，可 NULL |
| `code` | `string` | 部門代碼 |
| `name` | `string` | 部門名稱 |
| `description` | `string` | 部門說明 |
| `status` | `string` | 部門狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

`parent_id` 必須指向**同一家公司**的部門。

---

# 4. `employee_department_histories`

**用途：記錄員工部門異動歷史**

例如：

```text id="0n6m0k"
2024/01/01 → 2025/06/30
研發一處

2025/07/01 → 2026/02/28
研發二處

2026/03/01 → NULL
產品部
```

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 部門異動紀錄 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `department_id` | `uuid` | 部門 ID |
| `start_date` | `date` | 開始日期 |
| `end_date` | `date` | 結束日期，可 NULL |
| `reason` | `string` | 異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 5. `job_titles`

**用途：職稱資料**

支援：

```text id="7ahqbl"
系統預設職稱
+
公司自訂職稱
```

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職稱唯一識別碼 |
| `company_id` | `uuid` | 所屬公司，可 NULL；NULL 表示系統預設 |
| `code` | `string` | 職稱代碼 |
| `name` | `string` | 職稱名稱 |
| `description` | `string` | 職稱說明 |
| `is_system` | `boolean` | 是否系統預設 |
| `status` | `string` | 職稱狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

初始可以提供：

```text id="a7h4bf"
董事長
總經理
副總經理
協理
經理
副理
課長
組長
主任
專員
助理
工程師
資深工程師
資深專員
技術員
顧問
會計
人資
業務
行銷
設計師
產品經理
專案經理
```

這些屬於**初始資料**，不是 Schema。

---

# 6. `employee_job_title_histories`

**用途：員工職稱異動歷史**

例如：

```text id="2z0b6r"
工程師
↓
資深工程師
↓
產品經理
```

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職稱異動紀錄 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `job_title_id` | `uuid` | 職稱 ID |
| `start_date` | `date` | 開始使用日期 |
| `end_date` | `date` | 結束使用日期，可 NULL |
| `reason` | `string` | 異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 7. `employee_dependents`

**用途：扶養親屬資料**

只保存 HR / 報稅需要的資料，不保存電話、地址等無必要個資。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 扶養資料唯一識別碼 |
| `employee_id` | `uuid` | 員工 ID |
| `name` | `string` | 扶養親屬姓名 |
| `identity_number_encrypted` | `binary` | 身分證字號，加密 |
| `identity_number_hash` | `binary` | 身分證字號 Hash |
| `birthday_encrypted` | `binary` | 出生年月日，加密 |
| `relationship_code` | `integer` | 親屬關係代碼 |
| `is_student` | `boolean` | 是否在學 |
| `is_disabled` | `boolean` | 是否具身心障礙狀態 |
| `is_unable_to_work` | `boolean` | 是否無謀生能力 |
| `is_cohabiting` | `boolean` | 是否符合同居條件 |
| `effective_date` | `date` | 開始列入扶養日期 |
| `end_date` | `date` | 結束扶養日期，可 NULL |
| `status` | `string` | 扶養資料狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

### `relationship_code`

直接寫在欄位註釋：

```text id="e5a0vs"
1 = 配偶
2 = 父親
3 = 母親
4 = 子女
5 = 兄弟姊妹
6 = 祖父母
7 = 孫子女
8 = 其他
```

**不建立 `dependent_relationships` 資料表。**

---

# 目前 Schema 關係

```text id="p2c7yp"
companies
│
├── departments
│
├── job_titles
│
└── employees
      │
      ├── employee_dependents
      │
      └── employee_employments
              │
              ├── employee_department_histories
              │        └── departments
              │
              └── employee_job_title_histories
                       └── job_titles
```

---

## 這次刻意排除

### ❌ 薪資

這次不建立：

```text id="4qj1x4"
employee_salary_histories
```

薪資會另外規劃一套完整 Schema，包含基本薪資、薪資項目、津貼、獎金、扣款、計薪週期等。

### ❌ 打卡

之後獨立討論。

### ❌ 請假

之後獨立討論。

### ❌ 權限

前面已經獨立完成：

```text id="7q6gq0"
roles
permissions
role_permissions
```

### ❌ 部門主管

目前不處理，等員工與權限關係確定後再討論。

---

### 我認為目前這版的核心已經很清楚：

> **`employees` 管「人」；`employee_employments` 管「這個人的每一次任職」；部門與職稱透過 History 追蹤任職期間的變化。**

這樣即使員工離職後多年再回公司，也不會破壞過去的人事資料。

---

## 2026/08/22 21:09:03

### 使用者

employment_type_code   要確實放入Schema  
你不是說職稱還有職務，怎麼沒看到你提出來

### 助理

對，你抓得對。上一版有兩個地方我沒有貫徹我們前面討論的決定：

1. `employment_type_code` **必須正式列入 `employee_employments` Schema**。
2. 我前面指出「**職稱 ≠ 職務**」，但總結時卻只留下 `job_title`，這是不完整的。

我重新整理，而且這次把「**目前確定的 Schema**」和「**尚未決定是否需要的概念**」分清楚。

---

# 一、`employees`

**說明：**  
員工個人主檔，代表「這個人」。不放任職期間、部門、職稱、職務等具有歷史性的資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID |
| `employee_code` | `string` | 公司內員工編號 |
| `name` | `string` | 員工姓名 |
| `gender` | `string` | 性別代碼，不使用 ENUM |
| `identity_number_encrypted` | `binary` | 身分證字號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證字號 Hash，用於查詢比對 |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `phone_encrypted` | `binary` | 電話，加密儲存 |
| `email_encrypted` | `binary` | Email，加密儲存 |
| `address_encrypted` | `binary` | 居住地址，加密儲存 |
| `status` | `string` | 員工主資料狀態，不使用 ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

---

# 二、`employee_employments`

**說明：**  
記錄員工與公司的每一次任職關係。員工離職後重新回任，建立新的任職紀錄。

這是後面部門、職稱、職務等歷史資料的核心。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 任職紀錄唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 員工 ID |
| `employment_type_code` | `integer` | 任職類型代碼：`1=正職`、`2=兼職`、`3=約聘`、`4=派遣`、`5=工讀`、`6=臨時`、`7=顧問`、`8=實習` |
| `hire_date` | `date` | 該次任職到職日期 |
| `leave_date` | `date` | 該次任職離職日期，可 NULL |
| `leave_reason_code` | `integer` | 離職原因代碼，可 NULL；代碼定義直接寫於欄位註釋 |
| `status` | `string` | 該次任職狀態，不使用 ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

### 這裡正式確定：

```text id="z20m3r"
employment_type_code
```

是 Schema 正式欄位。

而且不用：

```text id="8d6q72"
ENUM
```

不用另外建立：

```text id="1z9f5x"
employment_types
```

目前固定代碼直接寫在 Schema 說明。

---

# 三、`job_titles`

這裡代表的是：

> **職稱 Title**

例如：

```text id="i0kz9m"
工程師
資深工程師
經理
副理
協理
```

**說明：**  
儲存系統預設與公司自訂的職稱。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職稱唯一識別碼 |
| `company_id` | `uuid` | 所屬公司 ID，可 NULL；NULL 表示系統預設職稱 |
| `code` | `string` | 職稱代碼 |
| `name` | `string` | 職稱名稱 |
| `description` | `string` | 職稱說明 |
| `is_system` | `boolean` | 是否為系統預設職稱 |
| `status` | `string` | 職稱狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

---

# 四、新增：`job_functions`

這就是我前面提到、但上一版漏掉的：

> **職務 Function / Job Function**

它和職稱不同。

例如：

| 職稱 | 職務 |
|---|---|
| 工程師 | 後端開發 |
| 工程師 | 前端開發 |
| 工程師 | DevOps |
| 資深工程師 | 系統架構 |
| 專員 | 招募 |
| 專員 | 薪酬管理 |
| 經理 | 研發管理 |

所以：

```text id="4ksm6y"
職稱 = 工程師
職務 = 後端開發
```

這兩個不能混成一個欄位。

### Schema：`job_functions`

**說明：**  
儲存系統預設及公司自訂的職務資料，用於描述員工實際負責的工作內容。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職務唯一識別碼 |
| `company_id` | `uuid` | 所屬公司 ID，可 NULL；NULL 表示系統預設職務 |
| `code` | `string` | 職務代碼 |
| `name` | `string` | 職務名稱 |
| `description` | `string` | 職務說明 |
| `is_system` | `boolean` | 是否為系統預設職務 |
| `status` | `string` | 職務狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

---

# 五、`employee_job_title_histories`

**說明：**  
記錄員工在每一次任職期間的職稱異動歷史。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 紀錄唯一識別碼 |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `job_title_id` | `uuid` | 職稱 ID |
| `start_date` | `date` | 開始使用日期 |
| `end_date` | `date` | 結束使用日期，可 NULL |
| `reason` | `string` | 職稱異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 六、新增：`employee_job_function_histories`

既然我們確定「職稱」和「職務」是兩個概念，那職務同樣需要追蹤歷史。

**說明：**  
記錄員工在每一次任職期間的職務異動歷史。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 紀錄唯一識別碼 |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `job_function_id` | `uuid` | 職務 ID |
| `start_date` | `date` | 開始擔任日期 |
| `end_date` | `date` | 結束擔任日期，可 NULL |
| `reason` | `string` | 職務異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 七、`employee_department_histories`

維持之前的設計。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 紀錄唯一識別碼 |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `department_id` | `uuid` | 部門 ID |
| `start_date` | `date` | 開始隸屬日期 |
| `end_date` | `date` | 結束隸屬日期，可 NULL |
| `reason` | `string` | 部門異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 八、`employee_dependents`

維持之前設計。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 扶養資料唯一識別碼 |
| `employee_id` | `uuid` | 員工 ID |
| `name` | `string` | 扶養親屬姓名 |
| `identity_number_encrypted` | `binary` | 身分證字號，加密 |
| `identity_number_hash` | `binary` | 身分證字號 Hash |
| `birthday_encrypted` | `binary` | 出生年月日，加密 |
| `relationship_code` | `integer` | 親屬關係代碼：`1=配偶`、`2=父親`、`3=母親`、`4=子女`、`5=兄弟姊妹`、`6=祖父母`、`7=孫子女`、`8=其他` |
| `is_student` | `boolean` | 是否在學 |
| `is_disabled` | `boolean` | 是否具身心障礙狀態 |
| `is_unable_to_work` | `boolean` | 是否無謀生能力 |
| `is_cohabiting` | `boolean` | 是否符合同居條件 |
| `effective_date` | `date` | 開始列入扶養日期 |
| `end_date` | `date` | 結束扶養日期，可 NULL |
| `status` | `string` | 扶養資料狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

---

# 現在整體變成

```text id="0w7t5x"
companies
│
├── departments
│
├── job_titles
│
├── job_functions
│
└── employees
      │
      ├── employee_dependents
      │
      └── employee_employments
              │
              ├── employee_department_histories
              │       └── departments
              │
              ├── employee_job_title_histories
              │       └── job_titles
              │
              └── employee_job_function_histories
                      └── job_functions
```

### 三個概念現在明確分開

```text id="u9m8h7"
職稱 Title
    ↓
「你在公司是什麼職稱？」
工程師、經理、主任

職務 Function
    ↓
「你實際負責什麼工作？」
後端開發、招募、薪酬管理

部門 Department
    ↓
「你隸屬哪個組織？」
研發部、研發一處、人資部
```

而三者都可以有自己的歷史。

---

## 薪資這次正式排除

目前人事 Schema **不包含任何薪資欄位與薪資歷史表**。

之後會另外設計：

```text id="2t3g0r"
薪資模組
```

這樣可以避免現在還沒定義薪資制度，就先把人事 Schema 綁死。

**我認為這一版才比較符合我們目前已經討論出來的方向。**

---

## 2026/08/22 21:10:04

### 使用者

在思考一下有沒有其他問題

### 助理

有，我再用「**真的拿去做企業 HR SaaS，之後資料量變大也不會後悔**」的角度重新檢查一次。

目前架構方向是對的，但我看到 **8 個值得現在就處理的問題**。其中有 3 個我認為是「現在就要改」，其他可以先記錄。

---

# 一、最重要：`employees.status` 其實可能不需要

目前：

```text id="c1h0x2"
employees.status
employee_employments.status
```

這兩個很容易產生矛盾。

例如：

```text id="4v5d6e"
employees.status = 啟用

employee_employments.status = 已離職
```

到底這個人算不算在職？

### 我的建議

`employees` 只代表：

> **這筆員工人事主檔是否還存在、是否可使用**

而「在職 / 離職」應該完全由：

```text id="u0p1n2"
employee_employments
```

管理。

因此：

### `employees`

可以把：

```text id="x6m7q8"
status
```

改成比較單純的：

```text id="k9r0s1"
record_status
```

例如：

```text id="a2b3c4"
1 = 正常
2 = 停用
```

而：

### `employee_employments`

才有：

```text id="d5e6f7"
status
```

例如：

```text id="g8h9i0"
1 = 任職中
2 = 已離職
3 = 留職停薪
```

這樣責任非常清楚。

---

# 二、`employee_employments` 缺少「任職編號 / 次數」概念

例如：

```text id="j1k2l3"
員工 E001

第一次任職
2020 → 2023

第二次任職
2025 → 現在
```

目前靠 UUID 就可以區分。

技術上沒問題。

但是 HR 使用者可能會看到：

```text id="m4n5o6"
任職紀錄 #1
任職紀錄 #2
```

因此我會建議增加：

```text id="p7q8r9"
employment_sequence
```

型態：

```text id="s0t1u2"
integer
```

說明：

> 同一員工的任職次序，第一次任職為 1，重新任職遞增。

這不是必要欄位，但對**人事管理畫面與歷史資料理解**很有價值。

---

# 三、`employment_type_code` 很重要，但還少「聘僱性質」

我們目前：

```text id="v3w4x5"
employment_type_code

1 = 正職
2 = 兼職
3 = 約聘
4 = 派遣
5 = 工讀
6 = 臨時
7 = 顧問
8 = 實習
```

這很好。

但是要注意：

> 「任職類型」和「勞動契約性質」不一定完全相同。

例如：

```text id="y6z7a8"
正職
定期契約
不定期契約
```

這可能會影響未來：

- 勞健保
- 離職
- 薪資
- 請假
- 出勤
- 勞動法規

不過因為我們現在**不做薪資與打卡**，我建議：

**現在先不要加。**

只把這個問題記錄下來，等之後討論「聘僱制度」時再處理。

---

# 四、部門、職稱、職務的「歷史起始日」可能互相打架

目前有：

```text id="b1c2d3"
部門：
2025/01/01 → 2025/06/30

職稱：
2025/03/01 → 2025/12/31

職務：
2025/02/01 → NULL
```

這完全可能。

但我們要確定一件事情：

> **部門、職稱、職務是不是允許獨立異動？**

我認為答案應該是：

**可以。**

例如：

```text id="e4f5g6"
1/1：轉到研發部
2/1：職務改成後端開發
3/1：升任資深工程師
```

所以目前分三張 History 是正確的。

不要把它們硬綁成一張「員工異動表」。

---

# 五、職稱與職務目前有一個需要重新思考的地方

我們現在：

```text id="h7i8j9"
job_titles
job_functions
```

這個概念我認同。

但是 `job_functions` 這個名稱可能需要再想一下。

因為我們實際想描述的是：

> **員工實際擔任什麼工作**

例如：

```text id="k0l1m2"
職稱：工程師
職務：後端工程師
```

或：

```text id="n3o4p5"
職稱：專員
職務：招募專員
```

這比較接近：

```text id="q6r7s8"
Job Position / Job Role
```

而不是一般系統設計裡的「Function」。

### 所以我建議

目前先不要急著正式定名 `job_functions`。

**Schema 可以保留這個概念，但名稱我們下一輪再確認。**

這個值得先釐清。

---

# 六、員工可能「同時」有多個職務

這個是我覺得很重要的一個問題。

例如：

```text id="t9u0v1"
職稱：工程師

職務：
- 後端開發
- DevOps
```

甚至：

```text id="w2x3y4"
職稱：經理

負責：
- 研發管理
- 專案管理
```

如果 `employee_job_function_histories` 設計成：

```text id="z5a6b7"
一個時間只能有一個 job_function
```

未來可能會卡住。

所以現在應該先問：

> **一名員工是否允許同時擔任多個職務？**

如果答案是「可以」，那現在的：

```text id="c8d9e0"
employee_job_function_histories
```

就可能需要改成：

```text id="f1g2h3"
employee_job_functions
```

一個員工可以同時有多筆。

這個我認為是目前**最值得先決定的業務規則之一**。

---

# 七、員工與部門不一定永遠只有一個部門

目前：

```text id="i4j5k6"
employee_department_histories
```

假設：

```text id="l7m8n9"
研發部 → 主要部門
```

沒問題。

但現實可能有：

```text id="o0p1q2"
主要部門：研發部
兼任部門：資訊部
```

或者：

```text id="r3s4t5"
總公司：研發部
專案：PMO
```

所以未來可能需要區分：

```text id="u6v7w8"
department_type
```

例如：

```text id="x9y0z1"
1 = 主要部門
2 = 兼任部門
```

**但我現在不建議直接加。**

因為這會牽涉到「員工是否允許跨部門任職」，應該等我們正式討論組織架構時一起決定。

---

# 八、`deleted_at` 要非常小心

目前很多表都有：

```text id="a2c4e6"
deleted_at
```

HR 系統跟一般 CRUD 系統不太一樣。

例如：

```text id="g8i0k2"
員工 2020 入職
2023 離職
```

這些資料可能涉及：

- 薪資
- 扣繳
- 勞保
- 稅務
- 出勤
- 請假
- 法規保存

所以**員工主檔不應該真的被刪掉**。

我會比較傾向：

```text id="m4o6q8"
employees
```

保留永久資料。

需要「停用」就使用：

```text id="s0u2w4"
record_status
```

而不是：

```text id="y6a8c0"
deleted_at
```

至於其他純設定型資料，例如：

```text id="e1g3i5"
job_titles
departments
```

保留 `deleted_at` 就比較合理。

---

# 我目前重新評估後的架構

我會把它整理成：

```text id="j7l9n1"
                    companies
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   departments      job_titles      job_positions
        │               │               │
        │               │               │
        └───────┬───────┴───────┬───────┘
                │               │
                ▼               ▼
             employees
                │
        ┌───────┴────────┐
        │                │
        ▼                ▼
   dependents       employments
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
        department     title       position
         history      history      history
```

其中我把之前的：

```text
job_functions
```

先標成：

```text
job_positions
```

只是**概念名稱暫定**，因為我認為「職務」的英文定義還值得我們下一輪確認。

---

## 目前我認為「已經可以定案」的

| 項目 | 狀態 |
|---|---|
| 員工主檔 | ✅ |
| 多次任職 | ✅ |
| 到職 / 離職獨立 | ✅ |
| 任職類型 `employment_type_code` | ✅ |
| 部門歷史 | ✅ |
| 職稱資料表 | ✅ |
| 職稱歷史 | ✅ |
| 職務概念 | ⚠️ 名稱與多職務規則待確認 |
| 扶養資料 | ✅ |
| 薪資 | ⏸️ 這次完全排除 |
| 打卡 | ⏸️ |
| 請假 | ⏸️ |
| 權限 | 已另外規劃 |

### 我現在最建議先解決的其實只有兩件事：

**① 職務能不能同時有多個？**

**② 「職務」這個概念到底是「職務（Position）」還是「工作內容（Function）」？**

這兩個一旦定下來，人事這一塊的核心 Schema 就相當穩了。

---

## 2026/08/22 21:14:16

### 使用者

employees.status   拿掉

任職編號 / 次數  沒必要
聘僱性質   可以
Job Position   可以
有多個職務   可以

員工與部門不一定永遠只有一個部門  不可以

### 助理

好，這樣就很清楚了。我依照你的決定調整，並把目前人事 Schema 的規則重新定下來。

## 已確定的調整

| 項目 | 決定 |
|---|---|
| `employees.status` | ❌ 移除 |
| 任職編號 / `employment_sequence` | ❌ 不建立 |
| 任職類型 | ✅ 保留 `employment_type_code` |
| 聘僱性質 | ✅ 新增 |
| Job Position | ✅ 採用 |
| 一名員工多個職務 | ✅ 支援 |
| 一名員工同時多個部門 | ❌ 不允許 |

---

# 1. `employees`

**說明：**  
員工個人主檔，只代表員工本人，不儲存任職期間、部門、職稱、職務等任職歷史資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID |
| `employee_code` | `string` | 公司內員工編號 |
| `name` | `string` | 員工姓名 |
| `gender` | `string` | 性別代碼，不使用 ENUM |
| `identity_number_encrypted` | `binary` | 身分證字號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證字號 Hash，用於查詢比對 |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `phone_encrypted` | `binary` | 電話，加密儲存 |
| `email_encrypted` | `binary` | Email，加密儲存 |
| `address_encrypted` | `binary` | 居住地址，加密儲存 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

### 已移除

```text id="x6d1k8"
status
```

員工是否在職，完全由 `employee_employments` 判斷。

---

# 2. `employee_employments`

**說明：**  
記錄員工與公司的每一次任職關係。員工離職後再次回任，建立新的任職紀錄。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 任職紀錄唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 員工 ID |
| `employment_type_code` | `integer` | 任職類型代碼：`1=正職`、`2=兼職`、`3=約聘`、`4=派遣`、`5=工讀`、`6=臨時`、`7=顧問`、`8=實習` |
| `employment_nature_code` | `integer` | 聘僱性質代碼；例如 `1=不定期契約`、`2=定期契約`，實際代碼可於正式定義聘僱制度時確認 |
| `hire_date` | `date` | 該次任職到職日期 |
| `leave_date` | `date` | 該次任職離職日期，可 NULL |
| `leave_reason_code` | `integer` | 離職原因代碼，可 NULL |
| `status` | `string` | 該次任職狀態，不使用 ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

### 注意

這裡：

```text id="g3h5j7"
employment_type_code
```

和：

```text id="k9m1n3"
employment_nature_code
```

是兩個不同概念。

例如：

```text id="p5r7t9"
任職類型：
正職

聘僱性質：
不定期契約
```

或：

```text id="v1x3z5"
任職類型：
約聘

聘僱性質：
定期契約
```

---

# 3. `departments`

**說明：**  
儲存公司的部門組織架構，支援上下層級與樹狀結構。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 部門唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID |
| `parent_id` | `uuid` | 上層部門 ID，可 NULL |
| `code` | `string` | 部門代碼 |
| `name` | `string` | 部門名稱 |
| `description` | `string` | 部門說明 |
| `status` | `string` | 部門狀態，不使用 ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

例如：

```text id="b2d4f6"
總經理室
└── 研發部
    ├── 研發一處
    └── 研發二處
```

---

# 4. `employee_department_histories`

這裡依你的決定：

> **一名員工同一時間只能屬於一個部門。**

所以這張表仍然保留歷史，但每個時間點只能有一個有效部門。

**說明：**  
記錄員工在每次任職期間的部門異動歷史。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 部門異動紀錄唯一識別碼 |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `department_id` | `uuid` | 部門 ID |
| `start_date` | `date` | 開始隸屬日期 |
| `end_date` | `date` | 結束隸屬日期，可 NULL |
| `reason` | `string` | 部門異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

例如：

```text id="h8j0l2"
2024/01/01 → 2025/06/30
研發一處

2025/07/01 → 2026/02/28
研發二處

2026/03/01 → NULL
產品部
```

**不允許：**

```text id="n4p6r8"
2026/03/01 → NULL
研發部

2026/03/01 → NULL
產品部
```

同一任職期間只能存在一個有效部門。

---

# 5. `job_titles`

**職稱**

例如：

```text id="t0v2x4"
工程師
資深工程師
經理
副理
協理
```

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職稱唯一識別碼 |
| `company_id` | `uuid` | 所屬公司 ID，可 NULL；NULL 表示系統預設職稱 |
| `code` | `string` | 職稱代碼 |
| `name` | `string` | 職稱名稱 |
| `description` | `string` | 職稱說明 |
| `is_system` | `boolean` | 是否為系統預設職稱 |
| `status` | `string` | 職稱狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

---

# 6. `employee_job_title_histories`

**說明：**  
記錄員工職稱的歷史異動。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職稱異動紀錄唯一識別碼 |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `job_title_id` | `uuid` | 職稱 ID |
| `start_date` | `date` | 開始使用日期 |
| `end_date` | `date` | 結束使用日期，可 NULL |
| `reason` | `string` | 職稱異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 7. `job_positions`

這次正式把「職務」定義為 **Job Position**。

**說明：**  
儲存系統預設及公司自訂的職務。職務代表員工實際擔任的工作職位，例如「後端工程師」、「招募專員」、「財務分析師」。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職務唯一識別碼 |
| `company_id` | `uuid` | 所屬公司 ID，可 NULL；NULL 表示系統預設職務 |
| `code` | `string` | 職務代碼 |
| `name` | `string` | 職務名稱 |
| `description` | `string` | 職務說明 |
| `is_system` | `boolean` | 是否為系統預設職務 |
| `status` | `string` | 職務狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

---

# 8. `employee_job_position_histories`

因為你確定：

> **一名員工可以同時有多個職務。**

所以這裡不能設計成「某個時間只能一個職務」。

例如：

```text id="c1e3g5"
員工：
職稱：資深工程師

同時擔任：
- 後端工程師
- DevOps
```

因此：

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職務紀錄唯一識別碼 |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `job_position_id` | `uuid` | 職務 ID |
| `start_date` | `date` | 開始擔任日期 |
| `end_date` | `date` | 結束擔任日期，可 NULL |
| `reason` | `string` | 職務異動原因 |
| `description` | `string` | 職務說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

因此同一時間可以：

```text id="i7k9m1"
employment_id = E001

Position A
2026/01/01 → NULL

Position B
2026/03/01 → NULL
```

這是合法的。

---

# 9. `employee_dependents`

維持原設計。

**說明：**  
儲存員工申報扶養親屬所需的人事與報稅相關資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 扶養資料唯一識別碼 |
| `employee_id` | `uuid` | 員工 ID |
| `name` | `string` | 扶養親屬姓名 |
| `identity_number_encrypted` | `binary` | 身分證字號，加密 |
| `identity_number_hash` | `binary` | 身分證字號 Hash |
| `birthday_encrypted` | `binary` | 出生年月日，加密 |
| `relationship_code` | `integer` | 親屬關係代碼：`1=配偶`、`2=父親`、`3=母親`、`4=子女`、`5=兄弟姊妹`、`6=祖父母`、`7=孫子女`、`8=其他` |
| `is_student` | `boolean` | 是否在學 |
| `is_disabled` | `boolean` | 是否具身心障礙狀態 |
| `is_unable_to_work` | `boolean` | 是否無謀生能力 |
| `is_cohabiting` | `boolean` | 是否符合同居條件 |
| `effective_date` | `date` | 開始列入扶養日期 |
| `end_date` | `date` | 結束扶養日期，可 NULL |
| `status` | `string` | 扶養資料狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可 NULL |

---

# 最終關係

```text id="o2q4s6"
companies
│
├── departments
│
├── job_titles
│
├── job_positions
│
└── employees
      │
      ├── employee_dependents
      │
      └── employee_employments
              │
              ├── employee_department_histories
              │       └── departments
              │
              ├── employee_job_title_histories
              │       └── job_titles
              │
              └── employee_job_position_histories
                      └── job_positions
```

其中有一個非常重要的規則現在確定：

```text id="u6w8y0"
一個 Employment
    │
    ├── 一個有效 Department
    │
    ├── 一個有效 Job Title
    │
    └── 一個以上有效 Job Position
```

也就是：

**部門：單一**

**職稱：單一**

**職務：可以多個**

這樣的模型我認為已經比前一版合理很多。薪資則完全從這一版拿掉，之後獨立設計。

---

## 2026/08/22 21:15:51

### 使用者

記錄起來

### 助理

已記錄。

後續我們繼續討論這套 HR SaaS 時，我會以這一版 **人事 Schema 與規則**作為基準，不會再自行改掉已經確定的設計。

特別記住：

- `employees.status` ❌
- `employment_sequence` ❌
- `employment_type_code` ✅
- `employment_nature_code` ✅
- Job Title ✅
- Job Position ✅
- **一人可同時多個 Job Position** ✅
- **一人同一時間只能有一個 Department** ✅
- 部門、職稱、職務都要保留歷史 ✅
- 薪資本次不納入 ✅
- 不使用 ENUM ✅
- 敏感資料使用 `binary/varbinary` 加密 + Hash 查詢設計 ✅
- 離職後重新任職建立新的 `employee_employments` ✅

---

## 2026/08/22 21:17:07

### 使用者

討論薪資這一塊

### 助理

可以。既然**人事基本資料已經獨立完成**，薪資我建議不要直接想「薪水一個欄位」，而是從「**薪資制度 → 薪資項目 → 員工薪資設定 → 每期計薪結果**」來設計。

而且薪資會直接牽涉到台灣的勞健保、所得稅、加班費、請假扣薪等，所以這一塊最好比人事 Schema 更嚴謹。

我建議先**只討論需求與資料模型，不碰程式架構**。

---

# 一、先確定薪資模組要解決什麼

最基本會包含：

```text
員工
 ↓
薪資設定
 ↓
每月計薪
 ↓
薪資明細
 ↓
應發
 ↓
扣款
 ↓
實發
```

例如：

```text
王小明

基本薪資       50,000
主管加給        5,000
交通津貼        2,000
-------------------
應發            57,000

勞保            1,000
健保              800
所得稅            500
其他扣款          200
-------------------
實發            54,500
```

但是這裡有一個非常重要的觀念：

> **「薪資設定」和「薪資結果」一定要分開。**

因為薪資會變。

---

# 二、第一張：員工薪資歷史

這就是你之前提到：

> 薪水要分開，因為會調薪，要知道每個時段的薪資。

我完全同意。

例如：

```text
2025/01/01
基本薪資 40,000

2025/07/01
基本薪資 43,000

2026/01/01
基本薪資 46,000
```

因此不能：

```text
employees.salary
```

而應該：

### `employee_salary_histories`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資紀錄 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `salary` | `decimal` | 基本薪資 |
| `salary_type_code` | `integer` | 薪資計算類型 |
| `start_date` | `date` | 生效日期 |
| `end_date` | `date` | 結束日期，可 NULL |
| `reason_code` | `integer` | 薪資異動原因 |
| `description` | `string` | 異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

### 例如

```text
王小明

2025/01/01
40,000

2025/07/01
43,000

2026/01/01
46,000
```

這樣歷史就完整保留。

---

# 三、但「薪資」不能只有基本薪資

這是薪資系統真正開始複雜的地方。

例如：

```text
基本薪資       50,000
主管加給        5,000
職務加給        3,000
交通津貼        2,000
伙食津貼        2,400
```

如果我們直接在 `employee_salary_histories` 加：

```text
base_salary
manager_allowance
transport_allowance
...
```

很快就會爆掉。

因為每家公司薪資項目都不同。

所以我建議：

# 四、建立「薪資項目」

### `salary_items`

**用途：定義公司可以使用哪些薪資項目。**

例如：

```text
基本薪資
主管加給
職務加給
交通津貼
伙食津貼
績效獎金
全勤獎金
加班費
勞保
健保
所得稅
```

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資項目 ID |
| `company_id` | `uuid` | 所屬公司 |
| `code` | `string` | 薪資項目代碼 |
| `name` | `string` | 薪資項目名稱 |
| `type_code` | `integer` | 項目類型 |
| `calculation_type_code` | `integer` | 計算方式 |
| `is_taxable` | `boolean` | 是否計入所得 |
| `is_insurable` | `boolean` | 是否計入投保薪資 |
| `is_active` | `boolean` | 是否啟用 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 五、薪資項目 `type_code`

我們同樣不使用 ENUM。

可以先定義：

```text
1 = 應發
2 = 扣款
```

但我會建議**先不要把代碼定死**。

因為後面可能會出現：

```text
應發
扣款
公司負擔
代扣代繳
```

所以這個我們可以下一步再定。

---

# 六、員工薪資不能直接綁 salary_items

例如：

```text
王小明
基本薪資 50,000
主管加給 5,000
交通津貼 2,000
```

另一個員工：

```text
李小華
基本薪資 42,000
交通津貼 2,000
```

所以需要：

### `employee_salary_items`

**用途：定義「這個員工目前有哪些薪資項目」。**

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工薪資項目 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `salary_item_id` | `uuid` | 薪資項目 ID |
| `amount` | `decimal` | 金額 |
| `calculation_type_code` | `integer` | 該員工此項目的計算方式 |
| `start_date` | `date` | 生效日期 |
| `end_date` | `date` | 結束日期，可 NULL |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

這樣：

```text
王小明
│
├── 基本薪資 50,000
├── 主管加給 5,000
└── 交通津貼 2,000
```

就可以個別管理。

---

# 七、這樣「調薪」就有兩種

例如：

### 基本薪資

```text
2025/01/01 → 40,000
2025/07/01 → 43,000
2026/01/01 → 46,000
```

### 主管加給

```text
2025/03/01 → 5,000
2026/04/01 → 8,000
```

所以其實：

> **薪資歷史應該是「薪資項目的歷史」，而不是只有基本薪資歷史。**

這點我會建議我們現在就修正前面的概念。

也就是未來可能不需要：

```text
employee_salary_histories
```

而是統一：

```text
employee_salary_items
```

搭配：

```text
start_date
end_date
```

來保存歷史。

---

# 八、真正「每月發多少錢」又是另一回事

這個一定要獨立。

例如：

```text
2026/08
王小明

基本薪資       50,000
主管加給        5,000
交通津貼        2,000
加班費          3,500
-------------------
應發            60,500

勞保            1,000
健保              800
所得稅            500
-------------------
實發            58,200
```

這筆資料不能直接從「目前薪資設定」重新計算就算了。

因為未來可能：

```text
2026/08 發薪後
公司調整薪資項目
```

如果沒有保存當時的計薪結果，歷史薪資就會被改變。

所以一定要有：

### `payrolls`

代表：

> 一個計薪週期。

以及：

### `payroll_details`

代表：

> 每個員工這一期的計薪結果。

---

# 九、整體薪資架構

我目前會建議：

```text
salary_items
      │
      │
      ▼
employee_salary_items
      │
      │
      ▼
    payroll
      │
      ▼
payroll_details
      │
      ├── 應發明細
      ├── 扣款明細
      └── 實發
```

再加上：

```text
employee_employments
        │
        └── employee_salary_items
```

---

# 十、這裡還有一個非常重要的問題

我們現在不要急著把 Schema 全定死。

因為薪資系統有一個核心問題：

> **薪資到底是「固定金額」，還是「公式計算」？**

例如：

### 固定金額

```text
交通津貼 = 2,000
```

### 按比例

```text
主管加給 = 基本薪資 × 10%
```

### 按天

```text
日薪 = 基本薪資 / 30
```

### 按時數

```text
加班費 = 時薪 × 加班時數 × 倍率
```

### 甚至公式

```text
績效獎金 =
基本薪資 × 績效比例
```

如果要做成 SaaS，我會強烈建議：

**不要把薪資項目設計死成「只有 amount」。**

---

## 所以我建議下一步先不要急著建完整 Schema

先把薪資的「薪資項目」定義清楚：

```text
1. 基本薪資
2. 津貼
3. 獎金
4. 加班費
5. 扣款
6. 勞保
7. 健保
8. 所得稅
9. 其他
```

然後我們再決定：

> **每一種薪資項目到底支援「固定金額、比例、公式、按時數、按天數」哪些計算方式。**

這個決定會直接影響後面的 `salary_items`、`employee_salary_items`、`payroll` Schema，所以我認為這一步比現在直接把所有資料表定死更重要。

---

## 2026/08/22 21:19:15

### 使用者

要設定薪資的資料表是?

### 助理

如果你問的是：

> **「員工目前的薪資設定，要存在哪一張表？」**

我建議核心就是：

### `employee_salary_items`

它負責記錄**某一位員工目前/歷史上有哪些薪資項目，以及各自的設定**。

例如：

```text
王小明
│
├── 基本薪資     50,000
├── 主管加給      5,000
├── 交通津貼      2,000
└── 職務加給      3,000
```

Schema：

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工薪資設定唯一識別碼 |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `salary_item_id` | `uuid` | 薪資項目 ID |
| `amount` | `decimal` | 金額或固定值 |
| `calculation_type_code` | `integer` | 計算方式代碼 |
| `start_date` | `date` | 生效日期 |
| `end_date` | `date` | 結束日期，可 NULL |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

但這裡要注意，**它不是單純「薪水表」**。

它的關係是：

```text
salary_items
    │
    │ 定義「這是什麼薪資項目」
    ▼
employee_salary_items
    │
    │ 定義「這個員工有多少」
    ▼
payroll
    │
    │ 實際計算
    ▼
payroll_details
```

例如：

### `salary_items`

```text
基本薪資
主管加給
交通津貼
績效獎金
```

### `employee_salary_items`

```text
王小明
基本薪資 → 50,000
主管加給 → 5,000
交通津貼 → 2,000
```

---

### 如果你問的是「薪資設定主檔」要不要另外一張？

我目前**不建議**再增加一張：

```text
employee_salaries
```

因為容易變成：

```text
employee_salaries
        ↓
employee_salary_items
```

多一層沒有必要。

目前我會定：

> **`employee_salary_items` = 員工薪資設定 + 歷史**

而：

> **`salary_items` = 公司定義有哪些薪資項目**

至於真正「某月算出來多少錢」，才是另一層：

```text
payrolls
payroll_details
```

這樣「**設定**」和「**計薪結果**」不會混在一起。

