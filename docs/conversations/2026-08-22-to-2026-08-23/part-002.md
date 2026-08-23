# 人資系統需求原始對話紀錄 — Part 002

> 來源：ChatGPT 對話「整理人資系統需求」  
> 時區：Asia/Taipei  
> 範圍：2026/08/22 20:31:35 ～ 2026/08/22 20:41:56  
> 說明：保留原始對話順序，未去除重複、舊版或後續已修正內容。

---

## 2026/08/22 20:31:35

### 使用者

可以

### 助理

好，那目前 **Company 公司的 Schema** 就以這個方向作為基準。

目前已確定的原則：

1. **SaaS 多公司**
2. `company_code` 與系統 `id` 分離
3. 公司 Code：
   - 公司：`統編 + 3碼流水號`
   - 個人：`YYYYMMDD + 3碼流水號`
4. Code 不使用 `-`
5. 流水號只增不減，不重複使用
6. 同一負責人可以擁有多家公司
7. 不使用 DB `ENUM`
8. 地址直接放在 `companies`
9. 地址只有：
   - 公司登記地址
   - 實際地址
   - 發票地址
10. 負責人、業務、會計統一放在 `company_contacts`
11. 敏感資料：
   - 加密資料 → `binary / varbinary`
   - Hash → `binary`，例如 SHA-256 為 32 bytes
12. 姓名目前維持明文
13. 目前**只討論 Company，不延伸其他 HR 模組**

接下來我們可以直接在這個基礎上逐欄檢查，看看 **Company 還缺哪些真正必要的需求資料**，而不是急著進入其他模組。

---

## 2026/08/22 20:32:00

### 使用者

你資料結構也要記錄下來，不然到時候就忘了

### 助理

可以，已經把目前確認的 **Company 資料結構與規則記錄下來**。

之後我們繼續設計時，我會以這一版為基準，不會重新假設欄位或把前面已確定的規則忘掉。

目前版本可以視為 **Company Schema v1**。

---

## 2026/08/22 20:34:43

### 使用者

再來討論角色、權限，可以建立什麼角色，每個角色有什麼權限，權限要區分大權限跟次權限，例如薪資管理，底下有統計薪資

### 助理

可以。這部分我建議不要直接把「角色」和「權限」混在一起，而是做成：

```text
角色 Role
   ↓
擁有哪些權限 Permission
   ↓
權限有「大權限」與「次權限」
```

例如：

```text
薪資管理
├── 查看薪資
├── 新增薪資
├── 修改薪資
├── 刪除薪資
├── 統計薪資
└── 匯出薪資
```

這樣未來可以做到很細的授權。

---

# 一、先定義「大權限」

我建議大權限先按照 HR 業務模組切。

目前可以有：

| 大權限 Code | 名稱 |
|---|---|
| `COMPANY` | 公司管理 |
| `EMPLOYEE` | 員工管理 |
| `DEPARTMENT` | 部門管理 |
| `ATTENDANCE` | 打卡管理 |
| `LEAVE` | 請假管理 |
| `PAYROLL` | 薪資管理 |
| `REPORT` | 報表管理 |
| `SYSTEM` | 系統管理 |

但這裡有一個地方要特別注意：

**大權限不是角色。**

例如：

```text
PAYROLL
```

代表「薪資管理這個權限群組」。

---

# 二、次權限

以你說的薪資為例：

```text
PAYROLL
│
├── VIEW
├── CREATE
├── UPDATE
├── DELETE
├── STATISTICS
└── EXPORT
```

對應：

| Code | 名稱 | 說明 |
|---|---|---|
| `VIEW` | 查看薪資 | 查看薪資資料 |
| `CREATE` | 新增薪資 | 建立薪資資料 |
| `UPDATE` | 修改薪資 | 修改薪資資料 |
| `DELETE` | 刪除薪資 | 刪除薪資資料 |
| `STATISTICS` | 統計薪資 | 查看薪資統計 |
| `EXPORT` | 匯出薪資 | 匯出薪資資料 |

所以完整 Permission 可以理解成：

```text
PAYROLL.VIEW
PAYROLL.CREATE
PAYROLL.UPDATE
PAYROLL.DELETE
PAYROLL.STATISTICS
PAYROLL.EXPORT
```

---

# 三、但我不建議所有模組都硬套 CRUD

這一點很重要。

例如：

### 打卡

```text
ATTENDANCE
├── VIEW
├── CREATE
├── UPDATE
├── DELETE
├── APPROVE
├── ADJUST
└── EXPORT
```

### 請假

```text
LEAVE
├── VIEW
├── CREATE
├── UPDATE
├── DELETE
├── APPROVE
├── REJECT
└── EXPORT
```

### 薪資

```text
PAYROLL
├── VIEW
├── CREATE
├── UPDATE
├── DELETE
├── CALCULATE
├── STATISTICS
├── APPROVE
├── EXPORT
└── PUBLISH
```

因為 HR 系統很多權限不是單純 CRUD。

---

# 四、角色怎麼設計？

角色是「一組權限」。

我會先提供**系統預設角色**。

例如：

## 超級管理員

```text
SUPER_ADMIN
```

全部權限。

---

## 公司管理員

```text
COMPANY_ADMIN
```

負責整間公司的 HR 系統管理。

例如：

```text
公司管理       全部
員工管理       全部
部門管理       全部
打卡管理       全部
請假管理       全部
薪資管理       全部
報表管理       全部
系統設定       部分
```

---

## HR

```text
HR
```

例如：

```text
員工管理
├── 查看
├── 新增
├── 修改

部門管理
├── 查看
└── 修改

請假管理
├── 查看
├── 審核
└── 統計

打卡管理
├── 查看
├── 修正
└── 統計
```

---

## 會計

```text
ACCOUNTING
```

例如：

```text
薪資管理
├── 查看
├── 計算
├── 統計
└── 匯出

報表
├── 查看
└── 匯出
```

---

## 主管

```text
MANAGER
```

例如：

```text
員工管理
└── 查看

打卡管理
├── 查看
└── 審核

請假管理
├── 查看
└── 審核

薪資管理
└── 不允許
```

這個角色很重要：

> **主管可以管理自己的部門，但不一定能看到全公司的資料。**

所以後面權限其實還會需要「資料範圍」。

---

# 五、這裡會出現一個更重要的需求：資料範圍

例如：

```text
主管 A
```

有：

```text
LEAVE.VIEW
```

但是他不能看到全公司所有人的請假。

只能：

```text
自己的部門
```

所以：

```text
Permission
```

其實至少有兩個概念：

```text
做什麼？
+
可以對誰做？
```

例如：

```text
LEAVE.VIEW
```

可以再有：

```text
ALL
DEPARTMENT
SELF
```

變成：

```text
LEAVE.VIEW
    scope = DEPARTMENT
```

主管就只能看自己部門。

---

# 六、所以角色 Schema

我們先不要急著做實際 DB，我會先把業務模型定成：

```text
Role
├── id
├── company_id
├── code
├── name
├── description
├── is_system
├── status
├── created_at
└── updated_at
```

角色例如：

```text
SUPER_ADMIN
COMPANY_ADMIN
HR
ACCOUNTING
MANAGER
EMPLOYEE
```

---

# 七、Permission Schema

```text
Permission
├── id
├── parent_code
├── code
├── name
├── description
├── status
├── created_at
└── updated_at
```

例如：

```text
PAYROLL
```

是父權限：

```text
PAYROLL.VIEW
PAYROLL.CREATE
PAYROLL.UPDATE
PAYROLL.DELETE
PAYROLL.CALCULATE
PAYROLL.STATISTICS
PAYROLL.APPROVE
PAYROLL.EXPORT
```

這樣你 UI 就可以直接做成樹狀：

```text
☑ 薪資管理
   ☑ 查看薪資
   ☑ 新增薪資
   ☑ 修改薪資
   ☐ 刪除薪資
   ☑ 計算薪資
   ☑ 統計薪資
   ☐ 核准薪資
   ☑ 匯出薪資
```

---

# 八、Role ↔ Permission

最後就是：

```text
Role
  │
  └── RolePermissions
          │
          └── Permission
```

所以：

```text
HR
│
├── EMPLOYEE.VIEW
├── EMPLOYEE.CREATE
├── EMPLOYEE.UPDATE
├── DEPARTMENT.VIEW
├── ATTENDANCE.VIEW
├── ATTENDANCE.ADJUST
├── LEAVE.VIEW
├── LEAVE.APPROVE
└── ...
```

---

# 九、我建議目前先定義這個權限層級

```text
大權限
│
├── 公司管理
├── 員工管理
├── 部門管理
├── 打卡管理
├── 請假管理
├── 薪資管理
├── 報表管理
└── 系統管理
```

下面才是：

```text
薪資管理
│
├── 查看
├── 新增
├── 修改
├── 刪除
├── 計算
├── 統計
├── 審核
├── 匯出
└── 發布
```

**而角色只是把這些權限組合起來。**

這個設計最大的好處是：未來客戶可以自己建立「自訂角色」。

例如某家公司想建立：

```text
薪資專員
```

只給：

```text
PAYROLL.VIEW
PAYROLL.CALCULATE
PAYROLL.STATISTICS
```

完全不需要修改程式或新增一種固定角色。

不過目前我們還需要先決定一個關鍵問題：**這些角色是「系統預設角色」，還是公司可以自己建立/修改角色？** 這會直接影響後面的 Role Schema。

---

## 2026/08/22 20:35:25

### 使用者

可以先不定一有什麼功能，先定義Schema

### 助理

可以，而且我認為這樣比較好。先把**角色與權限的資料結構**定下來，功能清單之後再填，不要現在把 `PAYROLL`、`ATTENDANCE` 這些業務內容寫死。

另外延續我們前面的原則：

- 不使用 `ENUM`
- 不把業務功能硬編碼在 Schema
- `string` / `uuid` / `binary` / `datetime` 等使用通用型態
- 角色與權限分離
- 大權限與次權限有階層關係

---

# 一、Role

### `roles`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 角色 ID |
| `company_id` | `uuid` | 所屬公司 |
| `code` | `string` | 角色代碼 |
| `name` | `string` | 角色名稱 |
| `description` | `string` | 角色說明 |
| `is_system` | `boolean` | 是否為系統預設角色 |
| `status` | `string` | 角色狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

例如未來可以有：

```text
code = HR
name = 人資
```

但現在**不預設有哪些角色**。

---

# 二、Permission

我建議 Permission 本身就可以表達「大權限 / 次權限」。

### `permissions`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 權限 ID |
| `parent_id` | `uuid` | 父權限 ID，可 NULL |
| `code` | `string` | 權限代碼 |
| `name` | `string` | 權限名稱 |
| `description` | `string` | 權限說明 |
| `permission_type` | `string` | 權限層級 |
| `status` | `string` | 權限狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 三、大權限 / 次權限

利用：

```text
parent_id
```

形成樹狀結構。

例如：

```text
權限 A
│
├── 權限 B
│
├── 權限 C
│
└── 權限 D
```

資料：

```text
id = 001
parent_id = NULL
name = 大權限
```

```text
id = 002
parent_id = 001
name = 次權限 A
```

```text
id = 003
parent_id = 001
name = 次權限 B
```

這樣就不需要在 Schema 裡面寫死：

```text
薪資管理
統計薪資
```

---

# 四、Role ↔ Permission

一個角色可以有很多權限。

一個權限也可以被很多角色使用。

所以是：

```text
Role N ↔ N Permission
```

建立：

### `role_permissions`

| 欄位 | 型態 | 說明 |
|---|---|---|
| `role_id` | `uuid` | 角色 ID |
| `permission_id` | `uuid` | 權限 ID |
| `created_at` | `datetime` | 建立時間 |

這張表本身其實不需要自己的 `id`。

可以把：

```text
role_id + permission_id
```

視為唯一組合。

---

# 五、目前資料關係

```text
                    Company
                       │
                       │
                       ▼
                     Roles
                       │
                       │ N
                       ▼
               RolePermissions
                       │
                       │ N
                       ▼
                  Permissions
                       │
                       │
                 parent_id
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          大權限                次權限
```

實際上 Permission 可以不只兩層。

例如：

```text
A
└── B
    └── C
        └── D
```

所以 `parent_id` 是比較好的設計。

---

# 六、我會再調整一個地方

其實 `permission_type` **可以不要**。

因為：

```text
parent_id = NULL
```

就代表第一層。

```text
parent_id != NULL
```

就是子權限。

如果未來支援多層：

```text
A
└── B
    └── C
```

根本不需要知道它是：

```text
BIG
SUB
```

它只需要知道自己的 Parent。

所以我比較推薦：

### `permissions`

| 欄位 | 型態 |
|---|---|
| `id` | `uuid` |
| `parent_id` | `uuid nullable` |
| `code` | `string` |
| `name` | `string` |
| `description` | `string` |
| `status` | `string` |
| `created_at` | `datetime` |
| `updated_at` | `datetime` |
| `deleted_at` | `datetime` |

這樣更乾淨。

---

# 七、目前先定義 3 張表

所以現在角色/權限先只確定：

```text
roles
role_permissions
permissions
```

關係：

```text
Company
   │
   └── roles
          │
          └── role_permissions
                    │
                    └── permissions
                           │
                           └── parent_id → permissions
```

**現在完全不決定有哪些角色，也不決定有哪些功能權限。**

之後我們只需要往 `permissions` 裡面逐步增加業務權限即可。

另外，這一版 Schema 我也建議跟前面的 **Company Schema v1** 一起記錄，避免後面繼續討論時遺失。

---

## 2026/08/22 20:37:15

### 使用者

每一個Schema  一定要有說明跟欄位說明

### 助理

對，這個我會當成這個專案的固定規則。

之後每一個 Schema 都固定包含：

### Schema 名稱
說明這張 Schema 是做什麼的、和哪些資料有關。

### 欄位

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 系統內部唯一識別碼 |
| `...` | `...` | 詳細說明 |

另外如果有：

- 主鍵
- 外鍵
- 唯一性
- Nullable
- 預設值
- 資料關聯
- 特殊資料處理

也會一起標註。

例如目前的 `permissions`，之後會寫成這種完整格式，而不是只有一張欄位表：

---

## Schema：`permissions`

**說明：**  
儲存系統中所有可被授權的權限項目。權限採階層式結構，透過 `parent_id` 建立父子關係，因此可以支援大權限、次權限，甚至未來更多層級的權限。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 權限的系統唯一識別碼，主鍵 |
| `parent_id` | `uuid` | 父權限 ID；最上層權限為 NULL |
| `code` | `string` | 權限唯一代碼 |
| `name` | `string` | 權限顯示名稱 |
| `description` | `string` | 權限用途與功能說明 |
| `status` | `string` | 權限目前狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

**關聯：**

```text
permissions.parent_id
        ↓
permissions.id
```

這個格式之後全部照這個標準走。

---

## 2026/08/22 20:37:47

### 使用者

納在呈現角色跟權限的Schema  我看看

### 助理

可以，先只呈現**角色與權限相關 Schema**，不帶入任何具體功能，例如薪資、打卡、請假目前都不定義。

---

# 角色與權限資料結構

整體關係先定義為：

```text id="j7e3x5"
Company
   │
   └── Roles
        │
        └── Role Permissions
                 │
                 └── Permissions
                        │
                        └── Parent Permission
```

一共先有 **3 個 Schema**：

1. `roles`
2. `permissions`
3. `role_permissions`

---

## Schema：`roles`

**說明：**  
儲存公司的角色定義。角色是權限的集合，可以讓公司建立不同的角色，再將不同權限配置給角色。

角色屬於公司，因此不同公司可以有自己的角色。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 角色的系統唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司的 ID，關聯 `companies.id` |
| `code` | `string` | 角色代碼，用於系統識別 |
| `name` | `string` | 角色名稱 |
| `description` | `string` | 角色用途與說明 |
| `is_system` | `boolean` | 是否為系統預設角色 |
| `status` | `string` | 角色目前狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

### 關聯

```text id="f6r0id"
roles.company_id
        ↓
companies.id
```

---

# Schema：`permissions`

**說明：**  
儲存系統中的權限項目。

權限採用**階層式結構**，透過 `parent_id` 建立父子關係。

因此可以支援：

```text id="c1qv1k"
大權限
├── 次權限
├── 次權限
└── 次權限
```

甚至未來需要更多層也可以：

```text id="y1f8e9"
A
└── B
    └── C
```

目前不在 Schema 裡定義「哪些是大權限、哪些是次權限」，只透過父子關係表示。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 權限的系統唯一識別碼，主鍵 |
| `parent_id` | `uuid` | 父權限 ID；最上層權限為 NULL |
| `code` | `string` | 權限代碼，用於系統識別 |
| `name` | `string` | 權限名稱 |
| `description` | `string` | 權限用途與說明 |
| `status` | `string` | 權限目前狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

### 關聯

```text id="u7xk4c"
permissions.parent_id
        ↓
permissions.id
```

例如：

```text id="3l4gqf"
權限 A
│
├── 權限 B
│
├── 權限 C
│
└── 權限 D
```

資料可能是：

```text id="8k2h8y"
A.parent_id = NULL
B.parent_id = A.id
C.parent_id = A.id
D.parent_id = A.id
```

---

# Schema：`role_permissions`

**說明：**  
建立角色與權限之間的關聯。

一個角色可以擁有多個權限，一個權限也可以被多個角色使用，因此這是一個 **Many-to-Many（多對多）** 關聯 Schema。

| 欄位 | 型態 | 說明 |
|---|---|---|
| `role_id` | `uuid` | 角色 ID，關聯 `roles.id` |
| `permission_id` | `uuid` | 權限 ID，關聯 `permissions.id` |
| `created_at` | `datetime` | 建立關聯的時間 |

### 唯一性

```text id="d9a5b8"
(role_id, permission_id)
```

同一個角色不能重複綁定相同權限。

---

# 三個 Schema 的關係

```text id="qk1m9x"
┌─────────────────┐
│    companies    │
│                 │
│ id              │
└────────┬────────┘
         │
         │ 1:N
         ▼
┌─────────────────┐
│      roles      │
│                 │
│ id              │
│ company_id      │
└────────┬────────┘
         │
         │ N:N
         ▼
┌─────────────────────┐
│   role_permissions  │
│                     │
│ role_id             │
│ permission_id       │
└──────────┬──────────┘
           │
           │ N:1
           ▼
┌─────────────────────┐
│     permissions     │
│                     │
│ id                  │
│ parent_id           │──┐
└─────────────────────┘  │
           ▲             │
           └─────────────┘
             父子權限
```

---

## 目前先不定義的東西

這一版**刻意不決定**：

- 有哪些角色
- 有沒有「管理員」
- 有沒有「HR」
- 有沒有「會計」
- 有哪些大權限
- 有哪些次權限
- 權限能做到什麼操作
- 權限資料範圍
- 員工如何綁定角色

目前只把**角色與權限的基礎資料結構**定下來。

另外，我會沿用前面確定的規則：**所有 Schema 都會有「Schema 說明 + 欄位 + 型態 + 欄位說明 + 關聯/特殊規則」**。

---

## 2026/08/22 20:39:12

### 使用者

角色權限的Schema  設計記錄起來

### 助理

已記錄，之後角色與權限的設計會以這個版本為基準。

目前已記錄的角色/權限 Schema：

```text
roles
permissions
role_permissions
```

並保留：

- `permissions.parent_id` 的階層式權限設計
- `roles ↔ permissions` 多對多
- 不使用 `ENUM`
- 不先限定具體角色或功能
- 每個 Schema 必須有**說明、欄位型態、欄位說明、關聯/特殊規則**

後面我們可以直接在這個基礎上繼續設計。

---

## 2026/08/22 20:40:49

### 使用者

再來討論部門，依樣只討論Schema  ，部門有分上下層級，可以曹狀層級，例如總經理室，下轄研發部，研發一處

### 助理

可以，部門這邊同樣先**只設計 Schema，不討論具體功能與權限**。

你描述的需求是典型的**樹狀階層部門**：

```text
總經理室
├── 研發部
│   ├── 研發一處
│   ├── 研發二處
│   └── 研發三處
├── 業務部
│   ├── 國內業務
│   └── 國外業務
└── 財務部
```

這種情況我會讓 `departments` 自己關聯自己。

---

# Schema：`departments`

**說明：**  
儲存公司的部門資料。每個部門隸屬於一家公司，並可透過 `parent_id` 建立上下層級，因此可以形成無固定層數的樹狀部門結構。

例如：

```text
總經理室
└── 研發部
    └── 研發一處
```

其中：

```text
總經理室.parent_id = NULL
研發部.parent_id = 總經理室.id
研發一處.parent_id = 研發部.id
```

---

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 部門系統唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID，關聯 `companies.id` |
| `parent_id` | `uuid` | 上層部門 ID，可為 NULL；最上層部門為 NULL |
| `code` | `string` | 部門代碼，用於系統識別 |
| `name` | `string` | 部門名稱 |
| `description` | `string` | 部門說明 |
| `status` | `string` | 部門狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 最後修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間，可為 NULL |

---

# 關聯

### 公司 → 部門

```text id="f2w7s9"
companies
    │
    └── departments
```

一家公司可以有多個部門：

```text
Company 1 ───── N Departments
```

---

### 部門 → 部門

```text id="c2m8vb"
departments.parent_id
        ↓
departments.id
```

因此可以無限向下建立：

```text
總經理室
│
├── 研發部
│   │
│   ├── 研發一處
│   │   ├── A組
│   │   │   └── 小組
│   │   └── B組
│   │
│   └── 研發二處
│
└── 業務部
```

Schema 本身不需要限制只能兩層或三層。

---

# 部門資料範例

假設：

```text
總經理室
```

資料：

```text
id = A
company_id = COMPANY001
parent_id = NULL
code = GM
name = 總經理室
```

研發部：

```text
id = B
company_id = COMPANY001
parent_id = A
code = RD
name = 研發部
```

研發一處：

```text
id = C
company_id = COMPANY001
parent_id = B
code = RD01
name = 研發一處
```

形成：

```text
總經理室
└── 研發部
    └── 研發一處
```

---

## 一個我建議現在就確定的規則

`parent_id` 必須和 `company_id` 屬於同一家公司。

也就是不能出現：

```text
公司 A
└── 研發部

公司的研發部 parent_id
    ↓
公司 B 的總經理室
```

部門階層只能存在於同一個 Company 裡。

另外目前**不加入部門主管、員工、職位等資料**，因為那些屬於下一階段。

目前部門就先維持一張：

```text id="y6xk84"
departments
```

透過 `parent_id` 解決樹狀階層。

---

## 2026/08/22 20:41:23

### 使用者

記錄起來

### 助理

已記錄。

目前已確定的核心 Schema 有：

```text id="qg4v0a"
companies
company_contacts

roles
permissions
role_permissions

departments
```

之後設計下一個 Schema 時，我會延續目前的格式與規則。

---

## 2026/08/22 20:41:49

### 使用者

再來討論建立員工，員工有姓名、性別、電話

### 助理

可以。先只討論 **Employee Schema**，不延伸到打卡、薪資、請假等。

不過員工除了你目前提出的：

- 姓名
- 性別
- 電話

我建議 Schema 先保留和公司、部門的基本關聯，因為這些是「員工屬於哪家公司、哪個部門」的核心資料。

---

# Schema：`employees`

**說明：**  
儲存公司的員工基本資料。每

