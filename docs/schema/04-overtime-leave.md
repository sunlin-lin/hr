# 加班、補休、請假與公司贈與假

## 加班

### `overtime_requests`

**註釋：** 員工加班申請，保存日期、起訖、分鐘、原因、班表關聯與流程狀態。

### `overtime_approvals`

**註釋：** 加班核准、拒絕、撤回等審核歷史。

### `overtime_compensations`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `1=加班費` | `原對話此處未重列` | 待核對 | 依原對話之欄位用途；未新增額外規則 |
| `2=補休` | `原對話此處未重列` | 待核對 | 依原對話之欄位用途；未新增額外規則 |

## 補休

### `compensatory_leave_credits`

**註釋：** 每筆來源產生的補休額度批次。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `原對話此處未重列` | 待核對 | 主鍵，資料唯一識別碼 |
| `company_id` | `原對話此處未重列` | 待核對 | 所屬公司外鍵 |
| `employee_id` | `原對話此處未重列` | 待核對 | 員工外鍵 |
| `employment_id` | `原對話此處未重列` | 待核對 | 任職紀錄外鍵 |
| `source_type_code` | `原對話此處未重列` | 待核對 | 依原對話之欄位用途；未新增額外規則 |

### `compensatory_leave_rate_snapshots`

**註釋：** 補休核發者所選計價基準及計算所需薪資／規則 Snapshot。後續調薪不得改變；到期轉薪使用原 Snapshot。

### `compensatory_leave_transactions`

**註釋：** 取得、預約／凍結、使用、取消返還、調整、撤銷、到期轉薪等不可變帳本。

### `compensatory_leave_allocations`

**註釋：** 一次補休使用實際分配到哪些額度批次。最早到期優先，可部分使用，取消原路返還。

規則：到期日當天仍可使用；到期剩餘一定轉薪資；薪資結算後不可直接修改歷史。

## 請假核心

### `leave_types`

假別定義；特休、福利假、補休彼此分離。

### `leave_type_rules`

假別法規／公司規則與有效期間。

### `leave_entitlements`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint/uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employee_id` | `FK` | 必填 | 員工外鍵 |
| `leave_type_id` | `FK` | 必填 | 假別外鍵 |
| `source_type_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |

### `leave_balances` / `leave_balance_transactions`

前者為當前餘額彙總／快取；後者為取得、使用、返還、調整及到期的完整帳本。原始 `entitled_minutes` 不因使用而 UPDATE。

### `leave_requests`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint/uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employee_id` | `FK` | 必填 | 員工外鍵 |
| `request_no` | `varchar(30)` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `status_code` | `integer` | 必填 | 流程或資料狀態代碼 |
| `reason` | `text` | 選填 | 原因 |

### `leave_request_details`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `PK` | 必填 | 主鍵，資料唯一識別碼 |
| `leave_request_id` | `FK` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `leave_type_id` | `FK` | 必填 | 假別外鍵 |
| `leave_date` | `date` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `start_time` | `time` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `end_time` | `time` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `requested_minutes` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `reason` | `text` | 選填 | 原因 |

### `leave_request_allocations`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `PK` | 必填 | 主鍵，資料唯一識別碼 |
| `leave_request_detail_id` | `FK` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `entitlement_type_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `entitlement_id` | `FK` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `allocated_minutes` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `created_at` | `datetime` | 必填 | 建立時間 |

### `leave_request_approvals` / `leave_request_documents` / `leave_events`

分別保存多層審核歷史、診斷／死亡／親屬等證明附件、以及結婚、死亡、生產、流產、職災等特殊事件來源。

## 公司贈與假

### `company_leave_grant_batches`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 主鍵，資料唯一識別碼 |
| `batch_no` | `varchar(30)` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `name` | `varchar(100)` | 必填 | 顯示名稱 |
| `leave_type_id` | `FK` | 必填 | 假別外鍵 |
| `pay_type_code` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `granted_minutes` | `integer` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `effective_from` | `date` | 必填 | 生效開始日 |
| `effective_to` | `date` | 必填 | 生效結束日 |
| `reason` | `text` | 必填 | 原因 |
| `created_by` | `FK` | 必填 | 建立者外鍵 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |

同批不得混用假別、薪資類型、額度或有效期；`granted_minutes > 0`；起訖日必填。

### `company_leave_grants`

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 主鍵，資料唯一識別碼 |
| `batch_id` | `FK` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `employee_id` | `FK` | 必填 | 員工外鍵 |
| `status_code` | `integer` | 必填 | 流程或資料狀態代碼 |
| `granted_by` | `FK` | 必填 | 依原對話之欄位用途；未新增額外規則 |
| `granted_at` | `datetime` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `cancelled_by` | `FK` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `cancelled_at` | `datetime` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `cancel_reason` | `text` | 選填 | 依原對話之欄位用途；未新增額外規則 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |

逐員工處理，個別失敗可單獨重試；公司直接核發，員工不可互贈；生效前可撤銷，已使用歷史不可抹除；到期或離職後不可使用但資料保留。


