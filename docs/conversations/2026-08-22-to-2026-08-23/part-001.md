# 人資系統需求原始對話紀錄 — Part 001

> 來源：ChatGPT 對話「整理人資系統需求」  
> 時區：Asia/Taipei  
> 範圍：2026/08/22 20:11:32 ～ 2026/08/22 20:30:36  
> 說明：保留原始對話順序，未去除重複、舊版或後續已修正內容。

---

## 2026/08/22 20:11:32

### 使用者

做一套人資系統、包含打卡、計薪、請假、權限、部門、
，目前不討論程式碼架構，指單純討論需求，這套打算做sass，所以一定會有N家公司，所以先從公司資料開始著手


公司需要哪些資料，
公司類型：個人、公司
編號(公司用統編、個人的話系統用身分證做編碼排序(這邊可以做一套演算法來保證不會重複))
負責人資料(姓名、身分證、出生年月日、電話)，除了姓名，剩餘資料做加密

你幫我整理，順便擬好schema

### 助理

可以。既然這套是 **SaaS、多家公司共用同一套系統**，我會建議一開始就把「公司」視為最上層的 **Tenant（租戶）**。

而且你現在先不談程式架構，我們只談**業務需求與資料模型**。

你目前提出的方向基本正確，但有一個地方我會調整：

> **不要把身分證字號本身拿來當系統編號。**

因為身分證是高度敏感資料，而且未來如果系統編號、URL、API、Log 等地方出現，就會有個資風險。

比較好的做法是：

- 公司：統編作為「外部法定識別碼」
- 個人：身分證作為「法定識別資料」
- 系統內部：另外產生不可推導的 `company_id`
- 身分證：**加密儲存 + Hash 做唯一性檢查**

---

# 一、先定義「公司」這個概念

我會把公司資料拆成幾個層次：

```text
Company（租戶）
│
├── 基本資料
│
├── 法定識別資料
│
├── 負責人
│
├── 聯絡資料
│
├── 地址
│
├── 公司設定
│
└── 狀態
```

未來再往下接：

```text
Company
│
├── Departments
│   └── Employees
│
├── Positions
│
├── Attendance
│
├── Leave
│
├── Payroll
│
├── Roles / Permissions
│
└── Settings
```

這樣很適合你的 HR SaaS。

---

# 二、公司基本資料

我建議至少有：

| 欄位 | 說明 |
|---|---|
| `id` | 系統內部 Company ID |
| `company_type` | 個人 / 公司 |
| `company_code` | 系統產生的公司代碼 |
| `tax_id` | 統一編號，公司才有 |
| `name` | 公司/商號名稱 |
| `short_name` | 公司簡稱 |
| `status` | 啟用、停用、刪除等 |
| `created_at` | 建立時間 |
| `updated_at` | 更新時間 |

### `company_type`

不要只做：

```text
個人
公司
```

我反而建議未來定義成：

```text
INDIVIDUAL       個人
COMPANY          公司/法人
```

因為台灣實務上未來可能碰到：

- 公司
- 行號
- 商號
- 個人工作室
- 自然人雇主

如果你現在只做「個人 / 公司」，未來很容易需要改 Schema。

甚至可以先保留：

```text
company_type
    INDIVIDUAL
    COMPANY
```

另外增加：

```text
legal_type
```

例如：

```text
CORPORATION
SOLE_PROPRIETORSHIP
PARTNERSHIP
INDIVIDUAL
OTHER
```

這樣彈性會比較大。

---

# 三、公司編號

這裡我建議**分成三個概念**。

### 1. `id`

系統內部 ID。

例如：

```text
01K3X8...
```

或者 UUID。

這個完全不代表公司身分。

---

### 2. `company_code`

系統自己的公司編號。

例如：

```text
CMP000001
CMP000002
CMP000003
```

這是給管理者看的。

---

### 3. `tax_id`

如果是公司：

```text
12345678
```

就是統一編號。

這個是法定識別資料，不應該拿來當系統 Primary Key。

---

### 個人

個人沒有統編時：

```text
company_type = INDIVIDUAL
tax_id = NULL
```

另外有：

```text
identity_number_encrypted
identity_number_hash
```

例如：

```text
身分證：
A123456789

↓ 加密

identity_number_encrypted

↓ Hash

identity_number_hash
```

Hash 用來：

- 判斷是否已經註冊
- 唯一性檢查
- 查詢

真正的身分證則只存在加密資料。

---

# 四、負責人資料

你目前規劃：

> 姓名、身分證、出生年月日、電話

我認為可以。

但我會把它獨立出去，不直接全部塞在 `companies`。

例如：

```text
companies
    │
    └── company_owners
```

原因是未來可能出現：

- 負責人變更
- 多位負責人
- 法定代理人
- 負責人歷史紀錄

所以不要把：

```text
owner_name
owner_id
owner_birthday
owner_phone
```

全部硬塞在 Company。

---

# 五、負責人 Schema

概念上：

```text
company_owners

id
company_id

name
identity_number_encrypted
identity_number_hash
birthday_encrypted
phone_encrypted

is_current
created_at
updated_at
```

其中：

### 姓名

可以明文：

```text
name
```

### 身分證

不要：

```text
identity_number
```

而是：

```text
identity_number_encrypted
identity_number_hash
```

### 出生年月日

如果你的要求是「除了姓名全部加密」，那：

```text
birthday_encrypted
```

### 電話

```text
phone_encrypted
```

---

# 六、為什麼身分證要「加密 + Hash」？

這個非常重要。

假設只有：

```text
identity_number_encrypted
```

你可以解密，但不好直接拿來做唯一性索引。

所以做：

```text
原始：
A123456789

             ┌── Encryption ──→ 加密內容
             │
A123456789 ──┤
             │
             └── Hash ─────────→ 固定 Hash
```

資料庫：

```text
identity_number_encrypted
identity_number_hash
```

例如：

```text
identity_number_encrypted = xxxx...
identity_number_hash      = 8f91......
```

然後：

```text
UNIQUE(identity_number_hash)
```

就能防止同一個人重複建立。

---

# 七、公司 Schema，我會先定成這樣

如果現在只討論「公司」這個第一階段，我會先建立這個模型：

### `companies`

| 欄位 | 類型概念 | 說明 |
|---|---|---|
| `id` | UUID/ULID | 系統內部 ID |
| `company_code` | string | 系統公司編號 |
| `company_type` | enum | 個人 / 公司 |
| `legal_type` | enum | 法律型態 |
| `tax_id` | string nullable | 統一編號 |
| `name` | string | 公司名稱 |
| `short_name` | string nullable | 公司簡稱 |
| `status` | enum | 狀態 |
| `created_at` | datetime | 建立時間 |
| `updated_at` | datetime | 更新時間 |
| `deleted_at` | datetime nullable | 軟刪除 |

---

### `company_owners`

| 欄位 | 類型概念 | 說明 |
|---|---|---|
| `id` | UUID/ULID | 負責人資料 ID |
| `company_id` | FK | 所屬公司 |
| `name` | string | 姓名 |
| `identity_number_encrypted` | text | 加密身分證 |
| `identity_number_hash` | string | 身分證 Hash |
| `birthday_encrypted` | text | 加密生日 |
| `phone_encrypted` | text | 加密電話 |
| `is_current` | boolean | 是否目前負責人 |
| `created_at` | datetime | 建立時間 |
| `updated_at` | datetime | 更新時間 |

---

# 八、公司聯絡資料

這個我建議不要直接塞進 `companies`。

因為公司未來可能有：

- 公司電話
- 公司手機
- 公司 Email
- HR Email
- 財務 Email
- 客服 Email

所以可以：

### `company_contacts`

```text
id
company_id
contact_type
name
phone
email
is_primary
created_at
updated_at
```

例如：

```text
公司主要聯絡人
    王小明
    04-12345678
    hr@example.com
```

---

# 九、公司地址

也建議獨立。

因為未來 HR 系統很容易遇到：

- 登記地址
- 公司地址
- 辦公室
- 分公司
- 工廠
- 台北辦公室
- 台中辦公室

所以：

### `company_addresses`

```text
id
company_id
address_type
postal_code
city
district
address
is_primary
created_at
updated_at
```

例如：

```text
REGISTERED     登記地址
OFFICE         辦公地址
FACTORY        工廠
OTHER          其他
```

這樣未來增加分公司會容易很多。

---

# 十、還有一個很重要的：公司設定

HR 系統一定會慢慢出現大量「公司層級設定」。

例如：

```text
上班時間
下班時間
午休時間
遲到規則
早退規則
加班規則
請假規則
薪資結算日
發薪日
年度假期
打卡規則
```

但**現在不要急著全部塞進 companies**。

未來應該拆：

```text
companies
    │
    ├── company_settings
    │
    ├── attendance_settings
    │
    ├── leave_settings
    │
    └── payroll_settings
```

這樣比較乾淨。

---

# 十一、SaaS 最重要的資料關係

你這個系統最上層我會定成：

```text
                    SaaS
                     │
        ┌────────────┼────────────┐
        │            │            │
      Company A    Company B    Company C
        │            │            │
      部門           部門          部門
        │            │            │
      員工           員工          員工
        │
 ┌──────┼───────┬────────┐
 │      │       │        │
打卡    請假    薪資     權限
```

**所有 HR 資料最終都必須能追溯到 Company。**

例如：

```text
company
   ↓
department
   ↓
employee
   ↓
attendance
```

所以 `attendance` 不應該只有：

```text
employee_id
```

通常還會考慮：

```text
company_id
employee_id
```

因為 SaaS 的資料隔離非常重要。

---

# 十二、第一版我建議先把 Company 定義完整

目前我們可以先把第一層定成：

```text
Company
│
├── 基本資料
│   ├── company_id
│   ├── company_code
│   ├── company_type
│   ├── legal_type
│   ├── tax_id
│   ├── name
│   ├── short_name
│   └── status
│
├── 負責人
│   └── CompanyOwner
│       ├── name
│       ├── identity_number
│       ├── birthday
│       └── phone
│
├── 聯絡資料
│   └── CompanyContact
│
├── 地址
│   └── CompanyAddress
│
└── 設定
    └── CompanySettings
```

而後面的 HR 核心模型再接：

```text
Company
│
├── Department
│   └── Employee
│       ├── Attendance
│       ├── Leave
│       ├── Payroll
│       └── Permission
│
├── Position
├── WorkSchedule
├── Holiday
├── LeavePolicy
└── PayrollPolicy
```

### 我特別建議你現在先確定 3 個原則

**① `company_id` 才是 SaaS 真正的 Tenant ID**

不要用統編當 Tenant ID。

**② 身分證永遠不要當系統 ID**

身分證是法定識別資料，不是系統識別碼。

**③ 敏感資料採「加密值 + Hash 值」**

例如：

```text
姓名 → 明文
身分證 → encrypted + hash
生日 → encrypted
電話 → encrypted
```

這樣之後做權限控制時，可以做到「HR 可以看到電話，但一般管理員不能看到身分證」。

---

如果按照這個方向繼續，我下一步會建議**先不要碰員工、打卡、計薪**，而是把「公司建立流程」完整定義出來，例如：

> **建立公司 → 公司類型 → 法定資料 → 負責人 → 聯絡資料 → 地址 → 公司基本設定 → 建立第一位管理員**

把這條流程定好後，再進入 **部門 → 員工**，整個 HR 系統的資料骨架會比較穩。

---

## 2026/08/22 20:22:16

### 使用者

同一個負責人會有多間公司，所以不應該用身分證來拒絕申請，我這邊覺得是用統編當code並判別搜尋出來，沒有一間時，就統編+001，如果已經有一間了，就統編+002

個人的話就用年月日+001

### 助理

對，這個需求我認同，而且這樣其實比「身分證唯一」更符合你這套 HR SaaS 的實際情境。

你這裡其實是在定義 **「公司顯示/業務編號」**，而不是定義資料庫唯一識別。

## 你的規則可以整理成這樣

### 1. 公司

以 **統編**作為前綴：

```text
統編 + 三碼流水號
```

例如統編：

```text
12345678
```

第一次建立：

```text
12345678-001
```

第二間：

```text
12345678-002
```

第三間：

```text
12345678-003
```

所以即使：

> 同一個負責人 = 多家公司

完全沒問題。

例如：

```text
王小明
│
├── 12345678-001  ABC有限公司
├── 12345678-002  XYZ有限公司
└── 12345678-003  小明商行
```

**身分證不負責判斷公司是否重複。**

---

## 2. 個人

個人沒有統編，就用：

```text
YYYYMMDD + 三碼流水號
```

例如：

```text
20260822-001
20260822-002
20260822-003
```

這樣同一天註冊的人可以自然區分。

例如：

```text
20260822-001  王小明
20260822-002  李小華
20260822-003  陳大明
```

而且**完全不需要把身分證放進這個 Code**。

---

# 我會把它定義成 `company_code`

所以前面的 Schema 我會修正。

```text
companies

id
company_code
company_type
tax_id
name
short_name
status
created_at
updated_at
deleted_at
```

其中：

```text
id
```

是系統內部真正的 Primary Key。

而：

```text
company_code
```

是業務上的公司編號。

---

# Code 產生規則

可以明確定義成：

### 公司

```text
{統編}-{三碼流水號}
```

```text
12345678-001
12345678-002
12345678-003
```

### 個人

```text
{建立日期 YYYYMMDD}-{三碼流水號}
```

```text
20260822-001
20260822-002
20260822-003
```

---

# 但是這裡有一個很重要的問題

你說：

> 沒有一間時，就統編+001，如果已經有一間了，就統編+002

我會稍微修改成：

> **找到該前綴目前最大的流水號 + 1**

而不是「搜尋有沒有一間」。

例如：

```text
12345678-001
12345678-002
12345678-003
```

下一個一定：

```text
12345678-004
```

即使 `12345678-002` 被刪除了，也**不要重新使用 002**。

所以：

```text
001
002
003
```

刪掉 002：

```text
001
003
```

下一間仍然：

```text
004
```

這對 HR SaaS 很重要，因為公司編號最好是**永久不重複**。

---

# 還有一個 SaaS 必須考慮的問題：同時建立

假設兩個管理員同時建立：

```text
12345678-004
```

兩個 Request 同時查：

```text
目前最大 = 003
```

然後兩個都產生：

```text
12345678-004
```

所以需求層面應該定義：

> **`company_code` 必須全域唯一。**

產生規則可以是：

```text
Prefix = 統編
Sequence = 該 Prefix 的下一個未使用流水號
```

而資料庫最終還是要保證：

```text
UNIQUE(company_code)
```

這是「需求規則」而不是現在討論程式架構，所以我們現在先記下這個業務規則即可。

---

# 我甚至會把這個欄位名稱再區分

我會建議：

```text
company_code
```

就是：

```text
12345678-001
```

而：

```text
tax_id
```

才是：

```text
12345678
```

所以：

| 欄位 | 範例 | 用途 |
|---|---|---|
| `id` | `01K...` | 系統內部識別 |
| `company_code` | `12345678-001` | SaaS 公司編號 |
| `tax_id` | `12345678` | 法定統編 |
| `company_type` | `COMPANY` | 公司/個人 |
| `name` | ABC有限公司 | 公司名稱 |

個人則：

| 欄位 | 範例 |
|---|---|
| `id` | `01K...` |
| `company_code` | `20260822-001` |
| `tax_id` | NULL |
| `company_type` | `INDIVIDUAL` |
| `name` | 王小明 |

---

## 目前公司需求可以先定稿成

```text
Company
├── 系統 ID
├── 公司編號
├── 公司類型
├── 法定識別資料
│   └── 統編（公司）
├── 公司名稱
├── 公司簡稱
├── 負責人
│   ├── 姓名
│   ├── 身分證（加密）
│   ├── 出生年月日（加密）
│   └── 電話（加密）
├── 聯絡資訊
├── 地址
├── 公司狀態
└── 建立/修改時間
```

而**身分證不做唯一限制**，這點我會從原本的設計中拿掉。

另外，你這個 `統編-001 / 統編-002` 的設計還有一個好處：**同一個統編可以對應多個 HR Tenant**。這對你未來處理「同一企業集團、多個法人」或「同一老闆管理多家公司」會很方便。

接下來我會建議我們先把 **「公司建立」這個 Entity 完整定義完**，包含「公司類型、統編規則、負責人、聯絡人、地址、公司狀態、公司編號規則」，確認後再進 **部門**。

---

## 2026/08/22 20:22:57

### 使用者

code不要用-

### 助理

可以，那就直接定義成**無分隔符號的純 Code**。

### 公司 Code

統編 + 三碼流水號：

```text
12345678001
12345678002
12345678003
```

規則：

```text
公司 Code = 統編(8碼) + 流水號(3碼)
```

例如：

| 統編 | 公司 | Code |
|---|---|---|
| 12345678 | ABC有限公司 | `12345678001` |
| 12345678 | XYZ有限公司 | `12345678002` |
| 12345678 | 小明商行 | `12345678003` |

---

### 個人 Code

年月日 + 三碼流水號：

```text
20260822001
20260822002
20260822003
```

規則：

```text
個人 Code = 建立日期(YYYYMMDD) + 流水號(3碼)
```

例如：

| 建立日期 | 個人 | Code |
|---|---|---|
| 2026/08/22 | 王小明 | `20260822001` |
| 2026/08/22 | 李小華 | `20260822002` |
| 2026/08/22 | 陳大明 | `20260822003` |

---

### 我會把目前規則正式定義為

```text
company_code

公司：
tax_id + 3碼流水號

個人：
YYYYMMDD + 3碼流水號
```

並且：

1. `company_code` **全域唯一**
2. 流水號只增不減
3. 刪除資料後不重複使用舊 Code
4. 負責人身分證**不參與 Code 產生**
5. 同一負責人可以擁有多個公司
6. `id` 與 `company_code` 分離
7. 統編本身仍另外儲存在 `tax_id`

這樣我覺得很合理。

---

## 2026/08/22 20:24:09

### 使用者

地址併入基本資料，地只有包含公司登記地址、實際地址、發票地址

### 助理

可以，這樣更簡潔。**地址不需要獨立成 Entity**，直接放在公司的基本資料裡，而且只定義三種地址。

我會把目前的 `Company` 需求整理成這個版本：

## Company 公司資料

### 1. 基本資料

| 欄位 | 說明 |
|---|---|
| `id` | 系統內部 ID |
| `company_code` | 系統公司 Code |
| `company_type` | 個人 / 公司 |
| `legal_type` | 法律型態 |
| `tax_id` | 統一編號，公司才有 |
| `name` | 公司/個人名稱 |
| `short_name` | 公司簡稱 |
| `registered_address` | 公司登記地址 |
| `actual_address` | 實際營業/辦公地址 |
| `invoice_address` | 發票地址 |
| `status` | 公司狀態 |
| `created_at` | 建立時間 |
| `updated_at` | 修改時間 |
| `deleted_at` | 軟刪除時間 |

---

## 2. 三種地址

### 公司登記地址

```text
registered_address
```

政府登記的地址。

### 實際地址

```text
actual_address
```

實際辦公、營業的地址。

### 發票地址

```text
invoice_address
```

發票相關地址。

這三個地址可以**各自不同**。

例如：

```text
公司登記地址：
台中市西屯區...

實際地址：
台中市南屯區...

發票地址：
台中市西屯區...
```

---

## 3. 地址資料本身

我建議不要只存一個 `address` 字串，至少拆成：

```text
postal_code
city
district
address
```

例如：

```text
registered_postal_code
registered_city
registered_district
registered_address
```

因此完整會變成：

```text
registered_postal_code
registered_city
registered_district
registered_address

actual_postal_code
actual_city
actual_district
actual_address

invoice_postal_code
invoice_city
invoice_district
invoice_address
```

這樣未來要做：

- 地址搜尋
- 區域統計
- 郵遞區號
- 發票列印
- 地址選擇

都比較方便。

---

## 目前 Company 可以整理成

```text
Company
│
├── 基本資料
│   ├── id
│   ├── company_code
│   ├── company_type
│   ├── legal_type
│   ├── tax_id
│   ├── name
│   ├── short_name
│   │
│   ├── 公司登記地址
│   ├── 實際地址
│   └── 發票地址
│
├── 負責人
│   ├── 姓名
│   ├── 身分證（加密）
│   ├── 出生年月日（加密）
│   └── 電話（加密）
│
├── 聯絡資訊
│
└── 狀態 / 系統資訊
```

另外我會建議一個小細節：**實際地址與發票地址可以提供「同公司登記地址」的選項**，但資料層面仍然可以把完整地址存下來，避免未來登記地址變更時把歷史資料一起改掉。

---

## 2026/08/22 20:25:53

### 使用者

不要用enum   不然到時候換資料庫就玩了

### 助理

對，我同意。尤其你現在是在做 **SaaS HR 系統的需求與資料模型**，我會避免把業務規則綁死在 DB 的 `ENUM`。

我們可以改成**字串 + 代碼表（Master Data）**。

例如不要：

```text
company_type ENUM('INDIVIDUAL', 'COMPANY')
```

而是：

```text
company_type varchar
```

搭配：

```text
company_types
```

資料：

| code | name | status |
|---|---|---|
| `INDIVIDUAL` | 個人 | 1 |
| `COMPANY` | 公司 | 1 |

未來要增加：

```text
SOLE_PROPRIETORSHIP
PARTNERSHIP
```

不用修改資料庫欄位型別。

---

### 我建議整套 HR 系統都遵守這個原則

**不要使用 DB ENUM。**

凡是這類會隨需求變動的東西：

- 公司類型
- 法律型態
- 公司狀態
- 員工狀態
- 部門類型
- 職位類型
- 請假類型
- 打卡類型
- 薪資類型
- 權限類型

全部使用：

```text
varchar / string
```

然後由系統的 **Master Data / Code Table** 管理。

例如請假：

```text
leave_types

code       name
ANNUAL     特休
SICK       病假
PERSONAL   事假
MARRIAGE   婚假
```

而 `employees` 只存：

```text
status = "ACTIVE"
```

不把 `ACTIVE` 寫死在資料庫 Schema 裡。

這樣未來即使從：

**PostgreSQL → MariaDB → MySQL → SQL Server**

資料模型都不會被 ENUM 綁住。

而且這個原則也很適合你這套 SaaS，因為**不同公司未來可能會有自己的代碼與規則**。

---

## 2026/08/22 20:26:24

### 使用者

現在只有先討論公司。不要納入其他

### 助理

對，先把範圍鎖死在 **Company 公司資料**，其他 HR 模組暫時完全不納入。

目前我們確認的需求可以整理成：

## Company Schema

### `companies`

| 欄位 | 說明 |
|---|---|
| `id` | 系統內部唯一 ID |
| `company_code` | 公司在系統中的唯一 Code |
| `company_type` | 公司 / 個人，字串 |
| `legal_type` | 法律型態，字串 |
| `tax_id` | 統一編號，公司才有 |
| `name` | 公司名稱 / 個人姓名 |
| `short_name` | 公司簡稱 |
| `registered_postal_code` | 公司登記地址郵遞區號 |
| `registered_city` | 公司登記地址縣市 |
| `registered_district` | 公司登記地址區 |
| `registered_address` | 公司登記地址 |
| `actual_postal_code` | 實際地址郵遞區號 |
| `actual_city` | 實際地址縣市 |
| `actual_district` | 實際地址區 |
| `actual_address` | 實際地址 |
| `invoice_postal_code` | 發票地址郵遞區號 |
| `invoice_city` | 發票地址縣市 |
| `invoice_district` | 發票地址區 |
| `invoice_address` | 發票地址 |
| `status` | 公司狀態，字串 |
| `created_at` | 建立時間 |
| `updated_at` | 修改時間 |
| `deleted_at` | 軟刪除時間 |

---

### `company_owners`

負責人獨立出來：

| 欄位 | 說明 |
|---|---|
| `id` | 負責人資料 ID |
| `company_id` | 所屬公司 |
| `name` | 姓名，明文 |
| `identity_number_encrypted` | 身分證，加密 |
| `identity_number_hash` | 身分證 Hash |
| `birthday_encrypted` | 出生年月日，加密 |
| `phone_encrypted` | 電話，加密 |
| `created_at` | 建立時間 |
| `updated_at` | 修改時間 |

### 目前 Code 規則

**公司：**

```text
統編 + 3碼流水號
```

例如：

```text
12345678001
12345678002
12345678003
```

**個人：**

```text
YYYYMMDD + 3碼流水號
```

例如：

```text
20260822001
20260822002
20260822003
```

並且：

- `company_code` 全域唯一
- 流水號只增不減
- 不因刪除而重用
- 負責人身分證不作為公司唯一條件
- 同一負責人可以有多家公司
- 不使用 DB `ENUM`
- `company_type`、`legal_type`、`status` 都使用一般字串
- 地址直接放 `companies`
- 地址只有 **登記地址、實際地址、發票地址**

目前就先到這裡，**不延伸到員工、部門、打卡、請假、計薪、權限**。

---

## 2026/08/22 20:28:28

### 使用者

聯絡人資料跟負責人資料都綁再一起，區分為負責人、業務、會計

### 助理

可以，這樣我反而覺得更合理。

因為「負責人、業務、會計」本質上都是**公司的聯絡窗口**，不需要拆成 `company_owners` 和 `company_contacts` 兩套資料。

我們把它統一成：

## `company_contacts`

一家公司可以有多個聯絡人，每個聯絡人有一個角色。

### 聯絡人角色

先用字串：

```text id="o2x1f7"
OWNER       負責人
SALES       業務
ACCOUNTING  會計
```

不使用 `ENUM`。

---

### Schema

| 欄位 | 說明 |
|---|---|
| `id` | 聯絡人 ID |
| `company_id` | 所屬公司 |
| `contact_type` | 聯絡人類型 |
| `name` | 姓名 |
| `identity_number_encrypted` | 身分證，加密 |
| `identity_number_hash` | 身分證 Hash |
| `birthday_encrypted` | 出生年月日，加密 |
| `phone_encrypted` | 電話，加密 |
| `email_encrypted` | Email，加密 |
| `created_at` | 建立時間 |
| `updated_at` | 修改時間 |

---

### 例如一家公司

```text id="s7xvqi"
ABC有限公司
│
├── OWNER
│   └── 王大明
│       身分證
│       出生年月日
│       電話
│
├── SALES
│   └── 李小華
│       電話
│       Email
│
└── ACCOUNTING
    └── 陳小美
        電話
        Email
```

這樣就很清楚。

---

### 不過有一個需求我建議現在就定下來

**負責人、業務、會計不一定只有一個。**

例如：

```text id="srm6tf"
ABC有限公司

OWNER
王大明

SALES
李小華
林小強

ACCOUNTING
陳小美
張小芳
```

所以 `company_contacts` 應該是一對多：

```text id="6h4b85"
Company 1 ─────── N CompanyContacts
```

而不是：

```text id="i7xw9y"
companies
    owner_name
    sales_name
    accounting_name
```

---

## 目前公司資料模型

```text id="4yww6n"
companies
│
├── 基本資料
│   ├── id
│   ├── company_code
│   ├── company_type
│   ├── legal_type
│   ├── tax_id
│   ├── name
│   ├── short_name
│   │
│   ├── 登記地址
│   ├── 實際地址
│   └── 發票地址
│
├── company_contacts
│   ├── 負責人
│   ├── 業務
│   └── 會計
│
└── 系統資訊
    ├── status
    ├── created_at
    ├── updated_at
    └── deleted_at
```

這樣目前「公司」這一層就相當乾淨了。

另外，**負責人的身分證不應該做唯一限制**，因為同一個人可以擔任多家公司負責人；`identity_number_hash` 的用途只應該是讓你能安全地比對「這個聯絡人的身分證是否相同」，而不是阻止他建立第二家公司。

---

## 2026/08/22 20:29:22

### 使用者

系統資訊直接綁在基本資料就好

### 助理

對，可以。既然目前只討論 `Company`，沒必要刻意把「系統資訊」再獨立成一層。

那目前可以收斂成 **兩個資料概念**：

```text id="9yq3cb"
Company
│
├── 基本資料
│   ├── 公司識別
│   ├── 公司名稱
│   ├── 地址
│   └── 系統資訊
│
└── 聯絡人
    ├── 負責人
    ├── 業務
    └── 會計
```

## `companies`

| 欄位 | 說明 |
|---|---|
| `id` | 系統內部唯一 ID |
| `company_code` | 系統公司 Code |
| `company_type` | 公司 / 個人 |
| `legal_type` | 法律型態 |
| `tax_id` | 統一編號，公司才有 |
| `name` | 公司名稱 / 個人姓名 |
| `short_name` | 公司簡稱 |
| `registered_postal_code` | 登記地址郵遞區號 |
| `registered_city` | 登記地址縣市 |
| `registered_district` | 登記地址區 |
| `registered_address` | 登記地址 |
| `actual_postal_code` | 實際地址郵遞區號 |
| `actual_city` | 實際地址縣市 |
| `actual_district` | 實際地址區 |
| `actual_address` | 實際地址 |
| `invoice_postal_code` | 發票地址郵遞區號 |
| `invoice_city` | 發票地址縣市 |
| `invoice_district` | 發票地址區 |
| `invoice_address` | 發票地址 |
| `status` | 公司狀態 |
| `created_at` | 建立時間 |
| `updated_at` | 修改時間 |
| `deleted_at` | 軟刪除時間 |

## `company_contacts`

| 欄位 | 說明 |
|---|---|
| `id` | 聯絡人 ID |
| `company_id` | 所屬公司 |
| `contact_type` | 負責人 / 業務 / 會計 |
| `name` | 姓名 |
| `identity_number_encrypted` | 身分證，加密 |
| `identity_number_hash` | 身分證 Hash |
| `birthday_encrypted` | 出生年月日，加密 |
| `phone_encrypted` | 電話，加密 |
| `email_encrypted` | Email，加密 |
| `created_at` | 建立時間 |
| `updated_at` | 修改時間 |

### 目前關係

```text id="b7v3r0"
companies
    │
    └── company_contacts
            ├── OWNER
            ├── SALES
            └── ACCOUNTING
```

而且 `company_contacts` 是 **一對多**，所以同一家公司可以有多個業務、會計，也可以保留負責人資料。

目前先維持這個範圍，不再往員工、部門、打卡等方向擴。

---

## 2026/08/22 20:29:51

### 使用者

擬型態也要飆出來

### 助理

對，這樣在討論 Schema 時會比較完整。我們先把**資料型態也一起定下來**，但仍然不綁定特定資料庫，例如不使用 `ENUM`、`SERIAL` 這類 DB 專屬型態。

我會先以通用型態來定義：

- `string`
- `integer`
- `boolean`
- `datetime`
- `text`
- `uuid`

目前 Company Schema：

## `companies`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 系統內部唯一 ID |
| `company_code` | `string` | 系統公司 Code，全域唯一 |
| `company_type` | `string` | 公司 / 個人 |
| `legal_type` | `string` | 法律型態 |
| `tax_id` | `string` | 統一編號，公司才有 |
| `name` | `string` | 公司名稱 / 個人姓名 |
| `short_name` | `string` | 公司簡稱 |
| `registered_postal_code` | `string` | 登記地址郵遞區號 |
| `registered_city` | `string` | 登記地址縣市 |
| `registered_district` | `string` | 登記地址區 |
| `registered_address` | `string` | 公司登記地址 |
| `actual_postal_code` | `string` | 實際地址郵遞區號 |
| `actual_city` | `string` | 實際地址縣市 |
| `actual_district` | `string` | 實際地址區 |
| `actual_address` | `string` | 實際地址 |
| `invoice_postal_code` | `string` | 發票地址郵遞區號 |
| `invoice_city` | `string` | 發票地址縣市 |
| `invoice_district` | `string` | 發票地址區 |
| `invoice_address` | `string` | 發票地址 |
| `status` | `string` | 公司狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

---

## `company_contacts`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 聯絡人唯一 ID |
| `company_id` | `uuid` | 所屬公司 ID |
| `contact_type` | `string` | 負責人 / 業務 / 會計 |
| `name` | `string` | 姓名 |
| `identity_number_encrypted` | `text` | 身分證，加密 |
| `identity_number_hash` | `string` | 身分證 Hash |
| `birthday_encrypted` | `text` | 出生年月日，加密 |
| `phone_encrypted` | `text` | 電話，加密 |
| `email_encrypted` | `text` | Email，加密 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

### Code 規則

公司：

```text
company_code = tax_id + 3碼流水號
```

例如：

```text
12345678001
12345678002
12345678003
```

個人：

```text
company_code = YYYYMMDD + 3碼流水號
```

例如：

```text
20260822001
20260822002
20260822003
```

### 一個小地方我會修正

`tax_id` 我建議也是 `string`，**不要用 integer**。

因為統編本質上是「識別碼」而不是數值，例如：

```text
00123456
```

如果當成 integer，就會變成：

```text
123456
```

這種資料應該一律當字串處理。

同理：

- 郵遞區號 → `string`
- 電話 → `string`
- 身分證 → `string`
- 統編 → `string`
- 公司 Code → `string`

這些都是**識別資料，不是拿來計算的數字**。

---

## 2026/08/22 20:30:29

### 使用者

在密的資料為什麼不用varbinary?

### 助理

可以用，而且如果我們現在是在認真設計「**資料庫 Schema**」，那我會更傾向把**加密後的資料存成 binary**，而不是 `text`。

我前面用 `text` 是站在「通用資料模型、不綁資料庫」的角度；但如果你問的是**實際資料庫欄位設計**，`VARBINARY` 確實更合理。

### 為什麼加密資料適合 `VARBINARY`

假設：

```text
身分證
A123456789
```

加密後本質上是：

```text
一串 bytes
```

不是文字。

所以：

```text
identity_number_encrypted VARBINARY
```

比：

```text
identity_number_encrypted TEXT
```

更符合資料本質。

如果你把 binary 轉成 Base64：

```text
原始 bytes
↓
Base64
↓
"8xK3...."
↓
TEXT
```

就會多一次編碼，而且資料量也會增加。

---

## 所以我們可以把 Schema 修正

### `company_contacts`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 聯絡人 ID |
| `company_id` | `uuid` | 所屬公司 |
| `contact_type` | `string` | 負責人 / 業務 / 會計 |
| `name` | `string` | 姓名 |
| `identity_number_encrypted` | `varbinary` | 身分證，加密 |
| `identity_number_hash` | `string` | 身分證 Hash |
| `birthday_encrypted` | `varbinary` | 出生年月日，加密 |
| `phone_encrypted` | `varbinary` | 電話，加密 |
| `email_encrypted` | `varbinary` | Email，加密 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

### `varbinary` 的長度

這個我反而不建議現在直接寫死：

```text
VARBINARY(255)
```

因為實際長度會跟你的**加密演算法、Nonce/IV、Tag、Key 版本、資料格式**有關。

需求層可以先定：

```text
varbinary
```

等真正決定加密方案時，再決定資料庫實際長度。

---

## Hash 就不一樣

Hash 的結果通常是固定長度 binary。

所以其實 `identity_number_hash` 也可以考慮：

```text
varbinary
```

例如 SHA-256：

```text
SHA-256 = 32 bytes
```

那麼：

```text
identity_number_hash VARBINARY(32)
```

會比：

```text
CHAR(64)
```

存十六進位字串更有效率。

所以如果我們追求**資料庫層面的正規設計**，我會改成：

```text
identity_number_encrypted  VARBINARY
identity_number_hash       VARBINARY(32)
birthday_encrypted         VARBINARY
phone_encrypted            VARBINARY
email_encrypted            VARBINARY
```

這個方向我認為比前面那版更好。

而且這也符合你前面說的「**不要被特定資料庫綁死**」——我們可以在需求 Schema 層定義成 `binary`，實際到不同 DB 再映射成該 DB 對應的 binary 型態。

