# 人資系統需求原始對話紀錄 — Part 003

> 來源：ChatGPT 對話「整理人資系統需求」  
> 時區：Asia/Taipei  
> 範圍：2026/08/22 20:42:02 ～ 2026/08/22 20:53:07  
> 說明：保留原始對話順序，未去除重複、舊版或後續已修正內容。

---

## 2026/08/22 20:42:02

### 使用者

再來討論建立員工，員工有姓名、性別、電話 ，你在幫我列舉

### 助理

可以。這一階段我們先把**員工基本資料需求**列完整，不先進入打卡、薪資、請假等功能。

你目前已經確定：

- 姓名
- 性別
- 電話

我會建議員工基本資料可以分成以下幾類。

# Schema：`employees`

**說明：**  
儲存公司員工的基本人事資料，以及員工所屬公司的基本關聯。員工屬於單一公司，後續再透過其他 Schema 延伸部門、職位、打卡、請假、薪資等資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工系統唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID，關聯 `companies.id` |
| `employee_code` | `string` | 員工在公司的識別編號 |
| `name` | `string` | 員工姓名 |
| `gender` | `string` | 性別，不使用 DB ENUM |
| `identity_number_encrypted` | `binary` | 身分證字號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證字號 Hash，用於比對 |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `phone_encrypted` | `binary` | 電話，加密儲存 |
| `email_encrypted` | `binary` | Email，加密儲存 |
| `address_encrypted` | `binary` | 居住地址，加密儲存 |
| `emergency_contact_name_encrypted` | `binary` | 緊急聯絡人姓名，加密儲存 |
| `emergency_contact_phone_encrypted` | `binary` | 緊急聯絡人電話，加密儲存 |
| `status` | `string` | 員工目前狀態 |
| `hire_date` | `date` | 到職日期 |
| `leave_date` | `date` | 離職日期，可為 NULL |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

---

## 我特別建議保留的資料

### 1. `employee_code`

這個很重要。

不要直接拿身分證當員工編號。

例如：

```text id="5r3fap"
EMP000001
EMP000002
EMP000003
```

這是公司內部員工識別碼。

而身分證則：

```text id="8s0k9q"
identity_number_encrypted
identity_number_hash
```

兩者分開。

---

### 2. 身分證

HR 系統通常會需要身分證，所以我建議一開始就放。

但跟前面的公司負責人一樣：

```text id="d4fj4x"
identity_number_encrypted
identity_number_hash
```

而且**不使用身分證當 Employee ID**。

---

### 3. 出生年月日

建議保留：

```text id="6w3qcm"
birthday_encrypted
```

因為人資系統後面很容易需要生日相關資料。

---

### 4. Email

建議有：

```text id="a1yq1q"
email_encrypted
```

但這裡可以之後再決定是不是需要加：

```text id="g1qg8m"
email_hash
```

如果未來 Email 可以作為登入帳號，就會需要另外處理唯一性。

---

### 5. 地址

員工住址屬於個資，因此我會建議：

```text id="f1m5bv"
address_encrypted
```

整體加密，不一定要像公司地址一樣拆縣市、區、路。

---

### 6. 緊急聯絡人

HR 系統通常會需要：

```text id="d5tw8p"
緊急聯絡人姓名
緊急聯絡人電話
```

目前先以一組為基礎即可。

---

# 目前我會把員工資料分成

```text id="4gqf4z"
Employee
│
├── 基本識別
│   ├── id
│   ├── company_id
│   └── employee_code
│
├── 個人資料
│   ├── name
│   ├── gender
│   ├── identity_number
│   ├── birthday
│   ├── phone
│   ├── email
│   └── address
│
├── 緊急聯絡
│   ├── contact_name
│   └── contact_phone
│
└── 任職資料
    ├── status
    ├── hire_date
    └── leave_date
```

### 但有兩個東西我**現在故意不放**

**部門**

你前面已經建立 `departments`，但我建議不要直接在這裡決定：

```text
employees.department_id
```

因為員工可能會：

> 研發部 → 研發一處 → 業務部

而且你未來可能需要**部門異動歷史**。

**角色 / 權限**

同樣先不要直接塞：

```text
employees.role_id
```

因為前面我們才剛把 Role / Permission Schema 建好，員工和角色之間的關係應該另外討論。

所以目前 `employees` 先把**「這個人是誰」以及「基本任職資訊」**處理好會比較乾淨。

---

## 2026/08/22 20:44:30

### 使用者

相關欄位
薪水，為什麼要分開，因為有調薪，分開才能知道每個時段的調薪

扶養資料

### 助理

對，這兩個都應該納入員工資料模型，但**薪水絕對不建議直接放在 `employees`**。

因為你說的原因很重要：

> 員工會調薪，所以薪資必須保留歷史。

「扶養資料」則建議也不要直接塞進 `employees`，因為一個員工可以有多筆扶養人。

我會把員工相關資料拆成：

```text id="qv7n8r"
employees
   │
   ├── employee_salary_histories
   │
   └── employee_dependents
```

---

# Schema：`employees`

**說明：**  
儲存員工目前的人事基本資料與任職資訊。薪資歷史與扶養資料另外以獨立 Schema 管理，避免將具有歷史性或一對多關係的資料直接放入員工主資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工系統唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID |
| `employee_code` | `string` | 員工識別編號 |
| `name` | `string` | 員工姓名 |
| `gender` | `string` | 性別 |
| `identity_number_encrypted` | `binary` | 身分證字號，加密 |
| `identity_number_hash` | `binary` | 身分證字號 Hash |
| `birthday_encrypted` | `binary` | 出生年月日，加密 |
| `phone_encrypted` | `binary` | 電話，加密 |
| `email_encrypted` | `binary` | Email，加密 |
| `address_encrypted` | `binary` | 居住地址，加密 |
| `status` | `string` | 員工狀態 |
| `hire_date` | `date` | 到職日期 |
| `leave_date` | `date` | 離職日期，可為 NULL |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

---

# Schema：`employee_salary_histories`

**說明：**  
儲存員工每一次薪資異動的歷史紀錄。每筆資料代表一個薪資生效期間，透過生效日期判斷不同時期的薪資，避免調薪後覆蓋舊資料。

例如：

```text id="l5qg8f"
2025/01/01 → 35,000
2025/07/01 → 38,000
2026/01/01 → 42,000
2026/06/01 → 45,000
```

歷史資料全部保留。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資歷史紀錄唯一識別碼 |
| `employee_id` | `uuid` | 員工 ID，關聯 `employees.id` |
| `salary` | `decimal` | 該期間的基本薪資 |
| `effective_date` | `date` | 薪資開始生效日期 |
| `end_date` | `date` | 薪資結束日期，可為 NULL；NULL 表示目前仍有效 |
| `reason` | `string` | 薪資異動原因 |
| `description` | `string` | 薪資異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

### 例如

```text id="6n4h2v"
employee_id = A
salary = 35000
effective_date = 2025-01-01
end_date = 2025-06-30
```

下一筆：

```text id="1cr0rf"
employee_id = A
salary = 38000
effective_date = 2025-07-01
end_date = 2025-12-31
```

再下一筆：

```text id="k4n4v4"
employee_id = A
salary = 42000
effective_date = 2026-01-01
end_date = NULL
```

這樣可以完整知道：

> **某個時間點，這個員工的薪資是多少。**

---

# Schema：`employee_dependents`

**說明：**  
儲存員工的扶養親屬資料。一名員工可以有多名扶養人，因此採一對多設計。每位扶養人獨立保存資料，並可記錄其與員工的關係及扶養狀態。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 扶養資料唯一識別碼 |
| `employee_id` | `uuid` | 所屬員工 ID |
| `name` | `string` | 扶養人姓名 |
| `relationship` | `string` | 與員工的關係 |
| `gender` | `string` | 性別 |
| `identity_number_encrypted` | `binary` | 身分證字號，加密 |
| `identity_number_hash` | `binary` | 身分證字號 Hash |
| `birthday_encrypted` | `binary` | 出生年月日，加密 |
| `is_dependent` | `boolean` | 是否目前列為扶養對象 |
| `effective_date` | `date` | 開始列入扶養日期 |
| `end_date` | `date` | 結束扶養日期，可為 NULL |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

---

## 為什麼扶養也要有日期？

因為扶養關係也會變化。

例如：

```text id="a6t8h4"
王小明
│
├── 王大寶
│   2025/01/01 開始扶養
│
├── 王小寶
│   2025/01/01 開始扶養
│   2026/03/01 結束扶養
│
└── 王小小
    2026/05/01 開始扶養
```

這樣未來查詢某個月份的資料時，就可以知道當時有哪些扶養人。

---

# 目前員工相關結構

```text id="0y8g8w"
employees
│
├── employee_salary_histories
│   └── 薪資歷史
│
└── employee_dependents
    └── 扶養資料
```

這裡有一個重要的設計原則：

**`employees` 放「現在這個人是誰」；有歷史性的資料就另外建立 History Schema。**

所以你提出的「薪水為什麼要分開」這個理由，我認為完全正確，而且這個原則後面設計其他員工資料時也值得沿用。

---

## 2026/08/22 20:45:39

### 使用者

在報稅上，扶養人需要提供什麼資料，我們只需要留需要的資料就好

### 助理

對，如果我們的目標是做 **台灣 HR SaaS**，而且「扶養資料」主要是為了後續**薪資、扣繳、所得稅申報**使用，那就不應該把一堆不需要的個資都存進來。

我查了目前財政部的申報資料。電子申報的扶養親屬基本資料主要包含：**姓名、身分證統一編號、出生年、稱謂**，另外依情況需要判斷 **在學、同居、無謀生能力** 等條件。citeturn0search17turn0search5

所以我會把我們前面的 `employee_dependents` 精簡。

# Schema：`employee_dependents`

**說明：**  
儲存員工申報扶養親屬所需的基本資料，以及判斷扶養資格所需的條件資料。資料以台灣綜合所得稅申報所需資訊為主要設計依據，避免儲存與扶養申報無關的個人資料。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 扶養資料唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 所屬員工 ID，關聯 `employees.id` |
| `name` | `string` | 扶養親屬姓名 |
| `identity_number_encrypted` | `binary` | 身分證統一編號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證統一編號 Hash，用於比對 |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `relationship` | `string` | 與員工的親屬關係 |
| `is_student` | `boolean` | 是否為在學狀態 |
| `is_disabled` | `boolean` | 是否具身心障礙狀態 |
| `is_unable_to_work` | `boolean` | 是否屬無謀生能力 |
| `is_cohabiting` | `boolean` | 是否符合同居條件 |
| `effective_date` | `date` | 開始列入扶養的日期 |
| `end_date` | `date` | 結束列入扶養的日期，可為 NULL |
| `status` | `string` | 扶養資料狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

### 為什麼保留這幾個條件？

因為不同扶養對象的資格條件不同。

例如財政部目前的規定中：

- 子女可能涉及**未成年、在學、身心障礙、無謀生能力**
- 兄弟姊妹可能涉及**未成年、在學、身心障礙、無謀生能力**
- 其他親屬則涉及**未成年、在學、身心障礙、無謀生能力**以及特定同居/扶養條件。citeturn0search5turn0search7

所以這些欄位不是單純「備註」，而是之後系統判斷申報資格可能會用到的資料。

---

## 但「證明文件」我不建議放這張表

例如：

```text
身心障礙證明
在學證明
親屬關係證明
戶籍資料
```

這些是**證明文件**，不是扶養人的基本資料。

之後如果我們真的做到「報稅文件管理」，再獨立做：

```text
employee_dependent_documents
```

這樣會比較乾淨。

---

## 另外一個重要調整

我會把之前的：

```text
gender
```

拿掉。

**因為目前我們的目標是「只保存報稅需要的資料」**，而財政部一般扶養親屬電子申報欄位並不需要我們另外保存性別；所以沒有必要為了「可能有用」就增加個資。citeturn0search17

同樣地：

```text
phone
email
address
```

也全部不放。

這樣 `employee_dependents` 就會比較符合你要的原則：

> **需要報稅才存，不需要就不要存。**

我會建議後面設計員工的其他資料時，也沿用這個原則：**不是 HR 常見資料就一定要存，而是先問「系統業務需要什麼」，再決定欄位。**

---

## 2026/08/22 20:46:38

### 使用者

親屬關係可以用選項

### 助理

可以，而且這裡我建議「**可以是選項，但資料庫不要用 ENUM**」，延續我們前面定的原則。

比較好的做法是 `relationship_code` 存一個 `string`，實際選項由系統資料管理。

## `employee_dependents` 調整

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 扶養資料唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 所屬員工 ID，關聯 `employees.id` |
| `name` | `string` | 扶養親屬姓名 |
| `identity_number_encrypted` | `binary` | 身分證統一編號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證統一編號 Hash |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `relationship_code` | `string` | 與員工的親屬關係代碼，由系統選項提供 |
| `is_student` | `boolean` | 是否為在學狀態 |
| `is_disabled` | `boolean` | 是否具身心障礙狀態 |
| `is_unable_to_work` | `boolean` | 是否屬無謀生能力 |
| `is_cohabiting` | `boolean` | 是否符合同居條件 |
| `effective_date` | `date` | 開始列入扶養日期 |
| `end_date` | `date` | 結束列入扶養日期，可為 NULL |
| `status` | `string` | 扶養資料狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

### UI 可以呈現

```text
親屬關係 ▼

○ 配偶
○ 父親
○ 母親
○ 子女
○ 兄弟姊妹
○ 祖父母
○ 孫子女
○ 其他
```

資料庫實際只存：

```text
relationship_code = "CHILD"
```

或：

```text
relationship_code = "SPOUSE"
```

而不是：

```text
relationship = "子女"
```

這樣未來如果：

- 改顯示名稱
- 增加選項
- 調整選項
- 不同語言顯示
- 配合報稅代碼

都不需要修改資料表結構。

**所以「UI 是選項」和「DB 不使用 ENUM」兩件事情完全不衝突。**

---

## 2026/08/22 20:47:46

### 使用者

用數字或代號代表你列的，並存入註釋中

### 助理

可以，這樣更符合我們目前的設計原則：

- DB 不使用 `ENUM`
- `relationship_code` 用固定代碼
- Schema 文件直接記錄「代碼 → 意義」
- 未來 UI 顯示文字可以獨立處理

我建議代碼不要直接用 `1、2、3`，而使用有意義的代碼，避免未來看資料時不知道 `3` 是什麼。

## Schema：`employee_dependents`

**說明：**  
儲存員工申報扶養親屬所需的資料，以及判斷扶養資格所需的條件。親屬關係使用 `relationship_code` 表示，不使用 DB ENUM。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 扶養資料唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 所屬員工 ID，關聯 `employees.id` |
| `name` | `string` | 扶養親屬姓名 |
| `identity_number_encrypted` | `binary` | 身分證統一編號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證統一編號 Hash |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `relationship_code` | `string` | 與員工的親屬關係代碼 |
| `is_student` | `boolean` | 是否為在學狀態 |
| `is_disabled` | `boolean` | 是否具身心障礙狀態 |
| `is_unable_to_work` | `boolean` | 是否屬無謀生能力 |
| `is_cohabiting` | `boolean` | 是否符合同居條件 |
| `effective_date` | `date` | 開始列入扶養日期 |
| `end_date` | `date` | 結束列入扶養日期，可為 NULL |
| `status` | `string` | 扶養資料狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

### `relationship_code` 代碼

| Code | 名稱 |
|---|---|
| `01` | 配偶 |
| `02` | 父親 |
| `03` | 母親 |
| `04` | 子女 |
| `05` | 兄弟姊妹 |
| `06` | 祖父母 |
| `07` | 孫子女 |
| `08` | 其他 |

例如資料：

```text id="xk11q8"
relationship_code = "04"
```

代表：

```text
04 = 子女
```

### Schema 註釋

```text
relationship_code:
01 = 配偶
02 = 父親
03 = 母親
04 = 子女
05 = 兄弟姊妹
06 = 祖父母
07 = 孫子女
08 = 其他
```

這樣之後實際建立資料庫 Schema 時，也可以把這些內容直接放進**欄位 COMMENT / DESCRIPTION** 裡。

不過有一點要注意：如果未來要直接對接台灣報稅申報格式，我們最後應該讓這些代碼**對應財政部正式的親屬/稱謂代碼**，而不是自己定義後直接拿去申報。這部分等我們做到「報稅」需求時再對照官方規格即可。

---

## 2026/08/22 20:48:30

### 使用者

我意思是，這邊員工的親屬關係代碼  要直接寫，不是再開一個資料表

### 助理

對，你的意思我懂了。

你要的是：

> **親屬關係是固定選項，所以直接定義在 Schema 的欄位註釋裡，不另外建立 `relationship` 資料表。**

這樣我同意，而且這個案例沒必要為了幾個固定選項再增加一張表。

---

# Schema：`employee_dependents`

**說明：**  
儲存員工申報扶養親屬所需的資料，以及判斷扶養資格所需的條件。親屬關係使用固定代碼儲存，不使用 DB ENUM，也不另外建立親屬關係資料表。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 扶養資料唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 所屬員工 ID，關聯 `employees.id` |
| `name` | `string` | 扶養親屬姓名 |
| `identity_number_encrypted` | `binary` | 身分證統一編號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證統一編號 Hash |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `relationship_code` | `string` | 與員工的親屬關係。`01=配偶`、`02=父親`、`03=母親`、`04=子女`、`05=兄弟姊妹`、`06=祖父母`、`07=孫子女`、`08=其他` |
| `is_student` | `boolean` | 是否為在學狀態 |
| `is_disabled` | `boolean` | 是否具身心障礙狀態 |
| `is_unable_to_work` | `boolean` | 是否屬無謀生能力 |
| `is_cohabiting` | `boolean` | 是否符合同居條件 |
| `effective_date` | `date` | 開始列入扶養日期 |
| `end_date` | `date` | 結束列入扶養日期，可為 NULL |
| `status` | `string` | 扶養資料狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

也就是資料庫只會有：

```text id="q4v5hs"
employee_dependents
```

**不會有：**

```text id="1b9m9j"
dependent_relationships
```

這種固定選項直接寫在 `relationship_code` 的欄位註釋中即可。

這個原則我也會沿用到後面的 Schema：**如果只是少量、穩定的固定選項，就直接在欄位說明中定義代碼，不為它額外建立資料表。**

---

## 2026/08/22 20:48:48

### 使用者

那為什麼不要用int型態

### 助理

可以用 `int`，而且在你現在這個情境，我反而認為 **`relationship_code` 用 `integer` 比 `string` 更合理**。

前面我用 `string` 是為了避免 `ENUM`，但「不用 ENUM」不代表一定要用字串。

### `relationship_code` 改成 `integer`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `relationship_code` | `integer` | 與員工的親屬關係代碼。`1=配偶`、`2=父親`、`3=母親`、`4=子女`、`5=兄弟姊妹`、`6=祖父母`、`7=孫子女`、`8=其他` |

資料庫：

```text
relationship_code = 4
```

代表：

```text
4 = 子女
```

### 為什麼我會選 `integer`

因為這個欄位的本質就是：

> **固定的數字代碼**

不是文字資料。

例如：

```text
1 = 配偶
2 = 父親
3 = 母親
4 = 子女
```

用 `integer` 有幾個好處：

- 不需要 `ENUM`
- 不需要建立另一張資料表
- 儲存空間小
- 查詢簡單
- 索引效率好
- 程式端很容易轉換成顯示文字
- Schema 仍然可以在欄位說明中完整定義代碼

---

### 那什麼時候才用 `string`？

例如：

```text
employee_code
company_code
tax_id
phone
postal_code
```

這些雖然可能長得像數字，但本質是**識別碼**。

例如：

```text
00123456
```

不能因為它只有數字就當 `integer`。

而：

```text
relationship_code = 4
```

這個 `4` 真的是一個**分類代碼**，所以 `integer` 很適合。

---

所以目前我會把它修正成：

```text
relationship_code : integer
```

而不是 `string`。

這個邏輯也可以套用到我們後面其他「固定代碼型欄位」：**不使用 ENUM，但如果代碼本身就是數字，就使用 integer；如果代碼是文字識別碼，就使用 string。**

---

## 2026/08/22 20:49:43

### 使用者

那重列一次員工及相關的Schema  我看一下

### 助理

可以。依照我們目前已經確定的規則，我把「**員工本身 + 員工相關資料**」重新整理一次。

目前只包含：

```text
employees
employee_salary_histories
employee_dependents
```

**不包含**部門、角色、權限、打卡、請假等關係。

---

# 1. Schema：`employees`

**說明：**  
儲存公司員工的基本人事資料與任職基本資訊。員工隸屬於單一公司。具有歷史性的資料，例如薪資調整，則另外建立獨立 Schema 保存。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工系統唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID，關聯 `companies.id` |
| `employee_code` | `string` | 員工在公司內的識別編號 |
| `name` | `string` | 員工姓名 |
| `gender` | `string` | 性別代碼；不使用 DB ENUM |
| `identity_number_encrypted` | `binary` | 身分證統一編號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證統一編號 Hash，用於比對 |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `phone_encrypted` | `binary` | 電話，加密儲存 |
| `email_encrypted` | `binary` | Email，加密儲存 |
| `address_encrypted` | `binary` | 居住地址，加密儲存 |
| `status` | `string` | 員工狀態代碼；不使用 DB ENUM |
| `hire_date` | `date` | 到職日期 |
| `leave_date` | `date` | 離職日期，可為 NULL |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

### 關聯

```text id="5u3n7q"
employees.company_id
        ↓
companies.id
```

---

# 2. Schema：`employee_salary_histories`

**說明：**  
儲存員工每次薪資異動的歷史資料。每筆資料代表一個薪資生效期間，避免調薪時直接覆蓋原本薪資，讓系統可以追溯員工在不同時間點的薪資。

例如：

```text id="1i3l5w"
2025-01-01 → 35,000
2025-07-01 → 38,000
2026-01-01 → 42,000
2026-06-01 → 45,000
```

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資歷史紀錄唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 所屬員工 ID，關聯 `employees.id` |
| `salary` | `decimal` | 該期間的基本薪資 |
| `effective_date` | `date` | 薪資開始生效日期 |
| `end_date` | `date` | 薪資結束日期，可為 NULL；NULL 表示目前仍有效 |
| `reason` | `string` | 薪資異動原因 |
| `description` | `string` | 薪資異動說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |

### 關聯

```text id="j7lq4a"
employees
   │
   └── employee_salary_histories
```

一名員工可以有多筆薪資歷史。

### 特殊規則

同一員工的薪資期間不應該互相重疊。

例如：

```text id="wl4n7x"
2025/01/01 ───── 2025/06/30   35,000
2025/07/01 ───── 2025/12/31   38,000
2026/01/01 ───── NULL          42,000
```

---

# 3. Schema：`employee_dependents`

**說明：**  
儲存員工申報扶養親屬所需的資料，以及判斷扶養資格所需的條件。親屬關係使用固定數字代碼，不使用 DB ENUM，也不另外建立親屬關係資料表。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 扶養資料唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 所屬員工 ID，關聯 `employees.id` |
| `name` | `string` | 扶養親屬姓名 |
| `identity_number_encrypted` | `binary` | 身分證統一編號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證統一編號 Hash，用於比對 |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `relationship_code` | `integer` | 親屬關係代碼：`1=配偶`、`2=父親`、`3=母親`、`4=子女`、`5=兄弟姊妹`、`6=祖父母`、`7=孫子女`、`8=其他` |
| `is_student` | `boolean` | 是否為在學狀態 |
| `is_disabled` | `boolean` | 是否具身心障礙狀態 |
| `is_unable_to_work` | `boolean` | 是否屬無謀生能力 |
| `is_cohabiting` | `boolean` | 是否符合同居條件 |
| `effective_date` | `date` | 開始列入扶養日期 |
| `end_date` | `date` | 結束列入扶養日期，可為 NULL |
| `status` | `string` | 扶養資料狀態代碼；不使用 DB ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

### 關聯

```text id="8s1o5k"
employees
   │
   └── employee_dependents
```

一名員工可以有多筆扶養資料。

---

# 三個 Schema 的整體關係

```text id="s0b0v9"
                         companies
                             │
                             │ 1:N
                             ▼
                        employees
                         │       │
                    1:N  │       │ 1:N
                         ▼       ▼
              salary_histories  dependents
```

也就是：

```text id="c0bqj9"
一家公司
   │
   └── 多名員工
          │
          ├── 多筆薪資歷史
          │
          └── 多名扶養親屬
```

---

## 目前特別遵守的設計原則

### ① 不用 ENUM

例如：

```text id="n0j0se"
gender
status
relationship_code
```

都不使用 DB `ENUM`。

固定數字代碼直接寫在**欄位說明**裡。

---

### ② 個資加密

需要保護的資料：

```text id="x1q8h0"
身分證
生日
電話
Email
地址
```

使用：

```text id="r9qk1s"
binary
```

儲存加密結果。

需要查詢比對的資料，再另外保存 Hash：

```text id="o6b0z5"
identity_number_hash
```

---

### ③ 薪資一定保留歷史

不在：

```text id="b7x0r1"
employees.salary
```

直接放目前薪資。

而是：

```text id="x8p4q6"
employee_salary_histories
```

讓每一次調薪都留下紀錄。

---

### ④ 扶養資料獨立

因為：

```text id="q4q6m5"
一個員工 → 多個扶養人
```

所以不把扶養人資料塞進 `employees`。

---

### ⑤ 目前不處理部門

雖然我們已經有：

```text id="k6n8x3"
departments
```

但目前刻意不加入：

```text id="x1b8s0p"
employees.department_id
```

因為「員工與部門」的關係還可以另外討論，尤其要考慮**調部門後的歷史紀錄**。

---

## 2026/08/22 20:52:08

### 使用者

員工資料表要有職稱，職稱可以另外建一個資料表，我們可以先塞入一批目前職場普遍的職稱進入

### 助理

可以。這樣「職稱」不要直接存文字在 `employees`，獨立成 `job_titles` 比較好。

而且因為這是 SaaS，我建議 **職稱資料表同時支援「系統預設職稱」與「公司自訂職稱」**。

例如系統預先提供：

```text
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
主管
會計
人資
業務
```

公司之後也可以自己增加：

```text
AI Agent 工程師
產品經理
技術長
...
```

---

# Schema：`job_titles`

**說明：**  
儲存系統提供及公司自訂的職稱資料。職稱可以由系統預設，也可以由個別公司自行建立。系統預設職稱可供所有公司使用，公司自訂職稱則僅限該公司使用。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 職稱系統唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID；NULL 表示系統預設職稱，非 NULL 表示公司自訂職稱 |
| `code` | `string` | 職稱代碼 |
| `name` | `string` | 職稱名稱 |
| `description` | `string` | 職稱說明 |
| `is_system` | `boolean` | 是否為系統預設職稱 |
| `status` | `string` | 職稱狀態，不使用 DB ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

### 關聯

```text id="t9l7wq"
job_titles.company_id
        ↓
companies.id
```

但：

```text id="6fj7d4"
company_id = NULL
```

代表這是**系統預設職稱**。

---

# `employees` 增加職稱

原本：

```text id="q1l3s5"
employees
```

增加：

| 欄位 | 型態 | 說明 |
|---|---|---|
| `job_title_id` | `uuid` | 員工職稱 ID，關聯 `job_titles.id`，可為 NULL |

所以員工相關欄位會變成：

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工系統唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID，關聯 `companies.id` |
| `employee_code` | `string` | 員工在公司內的識別編號 |
| `name` | `string` | 員工姓名 |
| `gender` | `string` | 性別代碼，不使用 DB ENUM |
| `job_title_id` | `uuid` | 員工職稱 ID，關聯 `job_titles.id` |
| `identity_number_encrypted` | `binary` | 身分證統一編號，加密儲存 |
| `identity_number_hash` | `binary` | 身分證統一編號 Hash |
| `birthday_encrypted` | `binary` | 出生年月日，加密儲存 |
| `phone_encrypted` | `binary` | 電話，加密儲存 |
| `email_encrypted` | `binary` | Email，加密儲存 |
| `address_encrypted` | `binary` | 居住地址，加密儲存 |
| `status` | `string` | 員工狀態代碼，不使用 DB ENUM |
| `hire_date` | `date` | 到職日期 |
| `leave_date` | `date` | 離職日期，可為 NULL |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

---

# 系統預設職稱

這部分我建議**先建立一批資料，但不要把這些資料寫死在 Schema**。

例如初始資料可以是：

| Code | 職稱 |
|---|---|
| `1001` | 董事長 |
| `1002` | 總經理 |
| `1003` | 副總經理 |
| `1004` | 協理 |
| `1005` | 經理 |
| `1006` | 副理 |
| `1007` | 課長 |
| `1008` | 組長 |
| `1009` | 主任 |
| `1010` | 專員 |
| `1011` | 助理 |
| `1012` | 工程師 |
| `1013` | 資深工程師 |
| `1014` | 資深專員 |
| `1015` | 技術員 |
| `1016` | 顧問 |
| `1017` | 會計 |
| `1018` | 人資 |
| `1019` | 業務 |
| `1020` | 行銷 |
| `1021` | 設計師 |
| `1022` | 產品經理 |
| `1023` | 專案經理 |

這些是**初始資料（seed data）**，不是資料結構的一部分。

---

## 有一個地方我建議現在就定下來

`job_title_id` **不要做成員工薪資歷史的一部分**。

因為：

```text
職稱 = 工程師
薪資 = 50,000
```

之後升遷：

```text
職稱 = 資深工程師
薪資 = 65,000
```

這其實是兩種不同的「歷史」：

```text
職稱異動歷史
薪資異動歷史
```

所以現在先：

```text
employees
   └── job_title_id → job_titles
```

等我們後面討論「**升遷 / 職稱異動**」時，再決定是否建立：

```text
employee_job_title_histories
```

這樣不會把現在的 `employees` 設計死。

---

## 2026/08/22 20:53:00

### 使用者

到職跟離職應該獨立出去，因為員工有可能離開後再回來

### 助理

對，這個判斷是對的。

如果 `hire_date`、`leave_date` 放在 `employees`，會假設一個員工只有一次任職期間。但實際上可能：

```text
第一次任職
2020/01/01 → 2023/06/30

離職

第二次任職
2025/03/01 → 目前
```

所以「員工」和「任職紀錄」應該拆開。

---

# Schema：`employees`

**說明：**  
儲存員工本人的基本資料。`employees` 代表「這個人」，不直接儲存每次任職的開始與結束時間。員工與公司的任職關係由 `employee_employments` 管理。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工系統唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID，關聯 `companies.id` |
| `employee_code` | `string` | 員工在公司內的識別編號 |
| `name` | `string` | 員工姓名 |
| `gender` | `string` | 性別代碼，不使用 DB ENUM |
| `job_title_id` | `uuid` | 目前職稱 ID，關聯 `job_titles.id` |
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

# Schema：`employee_employments`

**說明：**  
儲存員工與公司的每一段任職期間。每一次重新入職都建立一筆新的任職紀錄，因此可以完整保存員工多次任職、離職、再任職的歷史。

例如：

```text
員工 A

第一次任職
2020/01/01 ───── 2023/06/30

第二次任職
2025/03/01 ───── NULL
```

資料：

| id | employee_id | start_date | end_date |
|---|---|---|---|
| A001 | E001 | 2020-01-01 | 2023-06-30 |
| A002 | E001 | 2025-03-01 | NULL |

---

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 任職紀錄唯一識別碼，主鍵 |
| `employee_id` | `uuid` | 員工 ID，關聯 `employees.id` |
| `hire_date` | `date` | 該次任職的到職日期 |
| `leave_date` | `date` | 該次任職的離職日期，可為 NULL；NULL 表示目前仍在職 |
| `status` | `string` | 該次任職狀態，不使用 DB ENUM |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

---

# 但這會帶出一個重要問題

我們現在有：

```text
employees
    └── job_title_id
```

如果員工：

```text
工程師
↓
升遷
↓
資深工程師
↓
轉任
↓
主管
```

那麼 `employees.job_title_id` 只能知道**現在的職稱**，不知道歷史。

而且部門也會有同樣問題：

```text
研發部
↓
業務部
↓
管理部
```

所以我會建議我們接下來把「**員工任職資料**」獨立出來，未來可能形成：

```text
employees
    │
    └── employee_employments
             │
             ├── 部門
             ├── 職稱
             ├── 職務
             └── 任職期間
```

這樣會比把 `job_title_id` 直接放在 `employees` 更合理。

**因此我會建議現在連 `job_title_id` 也先不要放 `employees`，而是放到任職資料裡。**

這樣整體概念會變成：

```text
Employee
「這個人是誰」
       │
       ▼
Employment
「這個人在這家公司這一次怎麼任職」
       │
       ├── 到職
       ├── 離職
       ├── 部門
       └── 職稱
```

這個方向比較適合我們現在要做的完整 HR SaaS。

