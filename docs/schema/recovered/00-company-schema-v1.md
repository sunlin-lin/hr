# Company Schema v1（已定案恢復版）

> 狀態：已定案  
> 恢復依據：原需求對話中，在「目前版本可以視為 Company Schema v1」之前，使用者已同意的最後方案。  
> 範圍：只包含 Company；不延伸至部門、員工、打卡、薪資、請假或權限。  
> 原則：只記錄原對話已確認的欄位與規則；未確認的長度、預設值、索引或欄位不自行補入。

## 1. 整體定位

`companies` 是多公司 HR SaaS 的公司／租戶主體。系統內部識別、業務公司編號及法定統編彼此分離：

- `id`：系統內部主鍵，也是 Tenant 識別。
- `company_code`：系統使用的公司業務編號。
- `tax_id`：公司型主體的法定統一編號。
- 負責人、業務及會計統一存於 `company_contacts`。
- 公司登記地址、實際地址及發票地址直接存於 `companies`，不另建地址表。
- 不使用資料庫 ENUM；類型及狀態採一般字串欄位。
- 加密後資料以 binary／varbinary 概念保存；固定長度 Hash 以 binary 保存。

關聯：

```text
companies 1 ───── N company_contacts
```

## 2. `companies`

### 2.1 用途

保存 SaaS 中每一個公司或個人型租戶的基本資料、三組地址及系統狀態。公司系統資訊直接併入本表，不另拆表。

### 2.2 欄位

| 欄位 | 型態 | 必要性 | 說明 | 關聯／特殊規則 |
|---|---|---:|---|---|
| `id` | `uuid` | 必填 | 系統內部唯一識別碼、主鍵及 Tenant ID。 | 不使用 `company_code` 或 `tax_id` 作主鍵。 |
| `company_code` | `string` | 必填 | 系統公司 Code。 | 全域唯一；產生後不因刪除而重用。 |
| `company_type` | `string` | 必填 | 主體類型。 | 已討論值為 `COMPANY`、`INDIVIDUAL`；不使用 DB ENUM。 |
| `legal_type` | `string` | 必填 | 法律型態。 | 不使用 DB ENUM；v1 未定案完整代碼集合。 |
| `tax_id` | `string` | 視類型 | 統一編號。 | 公司型主體使用；個人型主體可為 NULL。識別碼使用字串，避免遺失前導零。 |
| `name` | `string` | 必填 | 公司正式名稱；個人型主體則為個人姓名。 | — |
| `short_name` | `string` | 選填 | 公司簡稱。 | — |
| `registered_postal_code` | `string` | 未定案 | 公司登記地址郵遞區號。 | 地址直接存於本表。 |
| `registered_city` | `string` | 未定案 | 公司登記地址縣市。 | — |
| `registered_district` | `string` | 未定案 | 公司登記地址行政區。 | — |
| `registered_address` | `string` | 未定案 | 公司登記詳細地址。 | — |
| `actual_postal_code` | `string` | 未定案 | 實際地址郵遞區號。 | 與登記地址分開保存完整值。 |
| `actual_city` | `string` | 未定案 | 實際地址縣市。 | — |
| `actual_district` | `string` | 未定案 | 實際地址行政區。 | — |
| `actual_address` | `string` | 未定案 | 實際營業／辦公詳細地址。 | — |
| `invoice_postal_code` | `string` | 未定案 | 發票地址郵遞區號。 | 與登記地址分開保存完整值。 |
| `invoice_city` | `string` | 未定案 | 發票地址縣市。 | — |
| `invoice_district` | `string` | 未定案 | 發票地址行政區。 | — |
| `invoice_address` | `string` | 未定案 | 發票詳細地址。 | — |
| `status` | `string` | 必填 | 公司目前狀態。 | 不使用 DB ENUM；v1 未定案完整狀態集合。 |
| `created_at` | `datetime` | 必填 | 建立時間。 | — |
| `updated_at` | `datetime` | 必填 | 最後修改時間。 | — |
| `deleted_at` | `datetime` | 選填 | Soft Delete 時間。 | NULL 表示未軟刪除。 |

> 地址欄位的「必要／可空」規則在原對話中未逐欄定案，因此本文件不自行指定。

### 2.3 `company_code` 規則

公司型主體：

```text
company_code = tax_id + 3 碼流水號
```

例如：

```text
12345678001
12345678002
```

個人型主體：

```text
company_code = YYYYMMDD + 3 碼流水號
```

例如同一天建立：

```text
20260822001
20260822002
```

已確認規則：

1. 不使用 `-` 或其他分隔符號。
2. `company_code` 全域唯一。
3. 流水號以相同前綴下目前最大值加一。
4. 流水號只增不減；資料刪除後不重用舊 Code。
5. `id` 與 `company_code` 分離。
6. `tax_id` 仍獨立保存，不以 `company_code` 取代。
7. 個人型主體的 Code 不使用身分證。
8. 同一負責人可以負責多家公司；負責人身分證不得用來拒絕建立另一家公司。
9. 資料庫需保證 `UNIQUE(company_code)`；實際併發取號機制尚未在 v1 定案。

### 2.4 地址規則

只保留三組地址：

- 公司登記地址
- 實際地址
- 發票地址

每組拆為郵遞區號、縣市、行政區及詳細地址，理由是支援地址搜尋、區域統計、郵遞區號處理、發票列印及地址選擇。

實際地址與發票地址在介面上可以提供「同公司登記地址」的操作，但資料層仍保存完整地址值，避免登記地址日後變更時連動改寫其他地址資料。

### 2.5 設計理由

- 以 `id` 作不可推導的內部主鍵，避免把統編或業務編號耦合到資料關聯。
- `company_code` 是穩定的業務識別；與法定統編分離後，同一統編可建立多個 HR Tenant。
- 三組地址是已確認範圍；直接放在 `companies` 可避免此階段不必要的地址 Entity。
- `company_type`、`legal_type`、`status` 使用一般字串，避免 DB ENUM 綁死後續需求。

## 3. `company_contacts`

### 3.1 用途

統一保存公司的負責人、業務及會計聯絡窗口。原先獨立的負責人概念已在定案前合併至本表，不建立 `company_owners`。

同一家公司可有多位同類型聯絡人，因此為一對多資料，不在 `companies` 上建立 `owner_name`、`sales_name` 或 `accounting_name` 等固定欄位。

### 3.2 欄位

| 欄位 | 型態 | 必要性 | 說明 | 關聯／特殊規則 |
|---|---|---:|---|---|
| `id` | `uuid` | 必填 | 聯絡人唯一識別碼、主鍵。 | — |
| `company_id` | `uuid` | 必填 | 所屬公司。 | FK → `companies.id`；一家公司可有多筆聯絡人。 |
| `contact_type` | `string` | 必填 | 聯絡人類型。 | 已討論值：`OWNER`、`SALES`、`ACCOUNTING`；不使用 DB ENUM。 |
| `name` | `string` | 必填 | 聯絡人姓名。 | v1 維持明文。 |
| `identity_number_encrypted` | `binary / varbinary` | 視聯絡類型／資料 | 加密後的身分證資料。 | 加密結果本質為 bytes；實際長度待加密方案確定。 |
| `identity_number_hash` | `binary` | 有身分證時 | 身分證查詢／安全比對用 Hash。 | 若採 SHA-256，可映射為 32 bytes；不得設跨公司唯一限制。 |
| `birthday_encrypted` | `binary / varbinary` | 視聯絡類型／資料 | 加密後的出生年月日。 | 實際長度待加密方案確定。 |
| `phone_encrypted` | `binary / varbinary` | 視聯絡類型／資料 | 加密後的電話。 | 實際長度待加密方案確定。 |
| `email_encrypted` | `binary / varbinary` | 視聯絡類型／資料 | 加密後的 Email。 | 實際長度待加密方案確定。 |
| `created_at` | `datetime` | 必填 | 建立時間。 | — |
| `updated_at` | `datetime` | 必填 | 最後修改時間。 | — |

> 原對話未逐項定案 OWNER、SALES、ACCOUNTING 各自必須填寫哪些敏感欄位，因此本文件只以「視聯絡類型／資料」表示，不自行新增必填規則。

### 3.3 聯絡類型

```text
OWNER       負責人
SALES       業務
ACCOUNTING  會計
```

以上使用一般字串 code，不使用資料庫 ENUM。v1 未限制每種類型只能一筆，因此同一家公司可有多位負責人、業務或會計。

### 3.4 敏感資料處理規則

1. 姓名維持明文。
2. 身分證、出生年月日、電話及 Email 以加密後 binary／varbinary 保存。
3. 加密欄位實際長度取決於加密演算法、Nonce／IV、Authentication Tag、Key Version 與封裝格式；v1 不寫死長度。
4. 需要安全比對的身分證另存固定長度 binary Hash。
5. `identity_number_hash` 用於查詢／比對，不用來阻止同一人關聯多家公司，也不設跨公司唯一限制。
6. 原定案未包含 `phone_hash` 或 `email_hash`，因此 v1 不新增這兩個欄位。
7. 加密或 Hash 的具體演算法與金鑰管理方式不屬於本次已定案 Schema，留待實作安全設計決定。

### 3.5 設計理由

- 負責人、業務與會計都是公司聯絡窗口，合併後避免兩套相似資料結構。
- 一對多設計支援同類型多位聯絡人。
- 敏感資料加密可降低資料外洩風險；Hash 讓系統無須解密即可比對身分證。
- Hash 不設跨公司唯一限制，符合「同一負責人可以擁有／負責多家公司」的業務規則。
- binary／varbinary 直接保存加密 bytes，避免 Base64 轉文字造成額外編碼與儲存膨脹。

## 4. 已確認約束與未定案事項

### 4.1 已確認

- `companies.id` 為主鍵／Tenant ID。
- `company_contacts.company_id` 關聯 `companies.id`。
- `companies 1:N company_contacts`。
- `UNIQUE(companies.company_code)`。
- 不使用 DB ENUM。
- 地址不拆表。
- 不建立 `company_owners`。
- 不以負責人身分證作公司唯一條件。
- Soft Delete 欄位為 `companies.deleted_at`。
- 敏感資料加密後使用 binary／varbinary；身分證 Hash 使用 binary。

### 4.2 v1 未定案，因此未自行補入

- 各 `string`、`binary / varbinary` 的實際長度。
- `company_type`、`legal_type`、`status` 的完整代碼集合。
- 三組地址的逐欄必填／可空條件。
- 每一種 `contact_type` 的欄位必填矩陣。
- 加密演算法、金鑰輪替、Nonce／IV／Tag 及 Key Version 的欄位封裝方式。
- `phone_hash`、`email_hash`。
- 主要聯絡人旗標、聯絡人狀態、軟刪除、有效期間或排序欄位。
- 公司設定、投保設定、部門、員工及其他 HR 模組欄位。

這些項目不得被視為 Company Schema v1 的既有定案；若需要加入，應另行討論並升版。
