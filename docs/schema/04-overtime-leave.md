# 加班、補休、請假與公司贈與假

## 加班

### `overtime_requests`

**註釋：** 員工加班申請，保存日期、起訖、分鐘、原因、班表關聯與流程狀態。

**設計理由：** 加班申請保存員工提出的日期、時數與原因，是流程來源；不直接以打卡超時當成已核准加班，避免出勤事實與申請資格混淆。


| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| — | — | — | 原對話已確認此資料表／資料責任，但此輪未能可靠恢復逐欄最終版本；不自行猜欄位。 |
### `overtime_approvals`

**註釋：** 加班核准、拒絕、撤回等審核歷史。

**設計理由：** 審核紀錄獨立成表，可保存多關卡、每次決定與時間，不以申請表上的單一狀態覆蓋審核歷程。


| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| — | — | — | 原對話已確認此資料表／資料責任，但此輪未能可靠恢復逐欄最終版本；不自行猜欄位。 |
### `overtime_compensations`

**資料表註釋：** `overtime_compensations` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 加班補償與申請分開，是因核准加班可選加班費、補休或其他已確認方式；補償是核准後的處置結果，不是申請本身。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| — | — | — | 已確認整筆核准加班只能選 1 加班費或 2 補休，不可拆分；此表逐欄最終版本未可靠恢復，因此不虛構欄位。 |

## 補休

### `compensatory_leave_credits`

**註釋：** 每筆來源產生的補休額度批次。

**設計理由：** 補休額度逐筆建立來源與到期日，才能追蹤每一筆加班轉換出的可用時數，並依來源處理使用及失效。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `型態待恢復` | 待核對 | 主鍵，資料唯一識別碼 |
| `company_id` | `型態待恢復` | 待核對 | 所屬公司外鍵 |
| `employee_id` | `型態待恢復` | 待核對 | 員工外鍵 |
| `employment_id` | `型態待恢復` | 待核對 | 任職紀錄外鍵 |
| `source_type_code` | `型態待恢復` | 待核對 | 欄位已確認；代碼值或額外約束未在定案節點明定 |

### `compensatory_leave_rate_snapshots`

**註釋：** 補休核發者所選計價基準及計算所需薪資／規則 Snapshot。後續調薪不得改變；到期轉薪使用原 Snapshot。

**設計理由：** 換算倍率保存快照，是為了讓額度沿用核准當時的法規或公司規則；日後倍率改變不應回算既有補休。


| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| — | — | — | 已確認存在；逐欄最終版本未可靠恢復。確認規則：最早到期優先、允許部分使用、申請中先凍結、取消原路返還。 |
### `compensatory_leave_transactions`

**註釋：** 取得、預約／凍結、使用、取消返還、調整、撤銷、到期轉薪等不可變帳本。

**設計理由：** 補休交易採不可只靠餘額的流水紀錄，可完整表達取得、使用、調整與到期，讓目前餘額能被稽核重建。


| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | PK |
| `code` | `varchar(30)` | 必填 | 假別代碼 |
| `name` | `varchar(50)` | 必填 | 假別名稱 |
| `category_code` | `integer` | 必填 | 1 法定、2 性別平等、3 公司福利、4 其他 |
| `is_paid` | `boolean` | 必填 | 是否有薪 |
| `requires_balance` | `boolean` | 必填 | 是否需要額度 |
| `requires_approval` | `boolean` | 必填 | 是否需要審核 |
| `requires_document` | `boolean` | 必填 | 是否要求證明文件 |
| `unit_code` | `integer` | 必填 | 1 日、2 小時、3 分鐘 |
| `is_active` | `boolean` | 必填 | 是否啟用 |
| `sort_order` | `integer` | 必填性待確認 | 顯示排序 |
| `description` | `text` | 選填 | 假別說明 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 修改時間 |
### `compensatory_leave_allocations`

**註釋：** 一次補休使用實際分配到哪些額度批次。最早到期優先，可部分使用，取消原路返還。

**設計理由：** 使用補休時另記額度分配，可指出一次請假實際扣到哪些來源額度，支援不同到期日與先到期先使用。

規則：到期日當天仍可使用；到期剩餘一定轉薪資；薪資結算後不可直接修改歷史。

## 請假核心


| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | PK |
| `leave_type_id` | `bigint` | 必填 | FK → `leave_types.id` |
| `rule_type_code` | `integer` | 必填 | 規則類型代碼 |
| `calculation_type_code` | `integer` | 必填 | 計算方式代碼 |
| `period_type_code` | `integer` | 選填 | 1 曆年、2 到職週年、3 月、4 事件、5 無固定期間 |
| `quota_minutes` | `integer` | 選填 | 標準額度分鐘數 |
| `max_quota_minutes` | `integer` | 選填 | 額度上限分鐘數 |
| `reference_leave_type_id` | `bigint` | 選填 | FK → `leave_types.id`，參照其他假別 |
| `eligibility_rule` | `json` | 選填 | 資格規則 |
| `quota_rule` | `json` | 選填 | 額度規則 |
| `salary_rule` | `json` | 選填 | 薪資給付規則 |
| `document_rule` | `json` | 選填 | 證明文件規則 |
| `effective_from` | `date` | 必填 | 生效日 |
| `effective_to` | `date` | 選填 | 失效日 |
| `is_active` | `boolean` | 必填 | 是否啟用 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 修改時間 |
### `leave_types`

**資料表註釋：** `leave_types` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 假別做成主檔，讓法定假、公司假及其他假別使用一致代碼與顯示設定，不把假別種類寫死在申請資料。

假別定義；特休、福利假、補休彼此分離。


| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| — | — | — | 原對話已確認此資料表／資料責任，但此輪未能可靠恢復逐欄最終版本；不自行猜欄位。 |
### `leave_type_rules`

**資料表註釋：** `leave_type_rules` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 假別規則與假別主檔分離，因同一假別的給付、單位、證明或限制可能隨政策調整，規則不應污染基本識別資料。

假別法規／公司規則與有效期間。


| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| — | — | — | 原對話已確認此資料表／資料責任，但此輪未能可靠恢復逐欄最終版本；不自行猜欄位。 |
### `leave_entitlements`

**資料表註釋：** `leave_entitlements` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 應享額度記錄員工在特定期間被授予的權利，將「制度規則」與「個人實際取得」分開，才能處理年資與個別調整。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint/uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employee_id` | `FK` | 必填 | 員工外鍵 |
| `leave_type_id` | `FK` | 必填 | 假別外鍵 |
| `source_type_code` | `integer` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `source_id` | `bigint` | 選填 | 來源資料 ID |
| `pay_type_code` | `integer` | 必填 | 1 有薪、2 無薪 |
| `entitled_minutes` | `integer` | 必填 | 授予分鐘數 |
| `effective_from` | `date` | 必填 | 可用起日 |
| `effective_to` | `date` | 選填 | 可用迄日 |
| `status_code` | `integer` | 必填 | 1 有效、2 撤銷、3 結清 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 修改時間 |

### `leave_balances`

**資料表註釋：** `leave_balances` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 餘額表提供快速查詢，交易表保留每次增減的可稽核來源；兩者分工可兼顧效能與可追溯性，不能只留一個可被直接改寫的數字。

前者為當前餘額彙總／快取；後者為取得、使用、返還、調整及到期的完整帳本。原始 `entitled_minutes` 不因使用而 UPDATE。


| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | PK |
| `employee_id` | `bigint` | 必填 | FK → `employees.id` |
| `leave_type_id` | `bigint` | 必填 | FK → `leave_types.id` |
| `entitled_minutes` | `integer` | 必填 | 已授予總分鐘數 |
| `reserved_minutes` | `integer` | 必填 | 申請中凍結分鐘數 |
| `used_minutes` | `integer` | 必填 | 已使用分鐘數 |
| `expired_minutes` | `integer` | 必填 | 已到期分鐘數 |
| `remaining_minutes` | `integer` | 必填 | 可用剩餘分鐘數；彙總／快取 |
| `updated_at` | `datetime` | 必填 | 最後重算時間 |

### `leave_balance_transactions`

**資料表註釋：** 假別額度不可變交易帳本，可由流水重建餘額。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | PK |
| `leave_entitlement_id` | `bigint` | 必填 | FK → `leave_entitlements.id` |
| `transaction_type_code` | `integer` | 必填 | 1 授予、2 凍結、3 使用、4 取消返還、5 結轉、6 到期、7 撤銷、8 人工調整 |
| `minutes` | `integer` | 必填 | 本次異動分鐘數 |
| `reference_type` | `varchar(50)` | 選填 | 來源資料類型 |
| `reference_id` | `bigint` | 選填 | 來源資料 ID |
| `occurred_at` | `datetime` | 必填 | 異動發生時間 |
| `created_by` | `bigint` | 必填 | 建立者 |
| `reason` | `text` | 選填 | 異動原因 |
| `created_at` | `datetime` | 必填 | 建立時間 |
### `leave_requests`

**資料表註釋：** `leave_requests` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 請假申請主表保存一次申請的共同資料與流程狀態，讓跨日或多日請假仍屬於同一申請。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint/uuid` | 必填 | 主鍵，資料唯一識別碼 |
| `employee_id` | `FK` | 必填 | 員工外鍵 |
| `request_no` | `varchar(30)` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `status_code` | `integer` | 必填 | 流程或資料狀態代碼 |
| `reason` | `text` | 選填 | 原因 |
| `applied_at` | `datetime` | 選填 | 送出申請時間 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 修改時間 |

### `leave_request_details`

**資料表註釋：** `leave_request_details` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 申請明細按日期／時段拆分，才能對照每天班表計算實際請假時數，避免以整段日曆時間誤算休息日。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `PK` | 必填 | 主鍵，資料唯一識別碼 |
| `leave_request_id` | `FK` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `leave_type_id` | `FK` | 必填 | 假別外鍵 |
| `leave_date` | `date` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `start_time` | `time` | 選填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `end_time` | `time` | 選填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `requested_minutes` | `integer` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `reason` | `text` | 選填 | 原因 |

### `leave_request_allocations`

**資料表註釋：** `leave_request_allocations` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 請假扣抵分配獨立保存，可追蹤申請使用哪一筆應享額度或補休來源，並支援單次申請跨多個額度。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `PK` | 必填 | 主鍵，資料唯一識別碼 |
| `leave_request_detail_id` | `FK` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `entitlement_type_code` | `integer` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `entitlement_id` | `FK` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `allocated_minutes` | `integer` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `created_at` | `datetime` | 必填 | 建立時間 |

### `leave_request_approvals`

**資料表註釋：** `leave_request_approvals` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 審核、附件與事件分表，是因三者分別代表流程決策、證明文件及狀態歷程；各自可能一對多，不能塞在申請主表的重複欄位中。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | PK |
| `leave_request_id` | `bigint` | 必填 | FK → `leave_requests.id` |
| `action_code` | `integer` | 必填 | 1 送出、2 核准、3 駁回、4 撤回、5 取消 |
| `action_by` | `bigint` | 必填 | 操作者 |
| `action_at` | `datetime` | 必填 | 操作時間 |
| `reason` | `text` | 選填 | 決定原因 |
| `created_at` | `datetime` | 必填 | 建立時間 |

### `leave_request_documents`

**資料表註釋：** 請假證明附件及驗證結果。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | PK |
| `leave_request_id` | `bigint` | 必填 | FK → `leave_requests.id` |
| `document_type_code` | `integer` | 必填 | 1 診斷、2 醫療、3 死亡、4 關係、5 其他 |
| `file_id` | `bigint` | 必填 | 附件檔案 ID |
| `verified_at` | `datetime` | 選填 | 驗證時間 |
| `verified_by` | `bigint` | 選填 | 驗證者 |
| `status_code` | `integer` | 必填 | 1 待驗、2 通過、3 未通過 |
| `created_at` | `datetime` | 必填 | 建立時間 |

### `leave_events`

**資料表註釋：** 結婚、親屬死亡、生產、流產、懷孕、配偶生產、職災等事件來源。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | PK |
| `employee_id` | `bigint` | 必填 | FK → `employees.id` |
| `event_type_code` | `integer` | 必填 | 1 結婚、2 親屬死亡、3 生產、4 流產、5 懷孕、6 配偶生產、7 職災、8 其他 |
| `event_date` | `date` | 必填 | 事件日期 |
| `relationship_code` | `integer` | 選填 | 親屬關係代碼 |
| `related_employee_id` | `bigint` | 選填 | 關聯員工 ID |
| `details` | `json` | 選填 | 事件補充資料 |
| `document_verified` | `boolean` | 必填 | 證明是否通過 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 修改時間 |

## 公司贈與假


| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| — | — | — | 原對話已確認此資料表／資料責任，但此輪未能可靠恢復逐欄最終版本；不自行猜欄位。 |
### `company_leave_grant_batches`

**資料表註釋：** `company_leave_grant_batches` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 公司贈與假先以批次表記錄發放原因、範圍與時間，讓一次對多人發放能被整批追蹤及稽核。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 主鍵，資料唯一識別碼 |
| `batch_no` | `varchar(30)` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `name` | `varchar(100)` | 必填 | 顯示名稱 |
| `leave_type_id` | `FK` | 必填 | 假別外鍵 |
| `pay_type_code` | `integer` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `granted_minutes` | `integer` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `effective_from` | `date` | 必填 | 生效開始日 |
| `effective_to` | `date` | 必填 | 生效結束日 |
| `reason` | `text` | 必填 | 原因 |
| `created_by` | `FK` | 必填 | 建立者外鍵 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |

同批不得混用假別、薪資類型、額度或有效期；`granted_minutes > 0`；起訖日必填。

### `company_leave_grants`

**資料表註釋：** `company_leave_grants` 的已確認資料責任；詳細規則依本節說明。

**設計理由：** 個別贈與結果另存每位員工實際取得的假別與額度，避免批次條件等同於最終結果，也便於處理個別失敗或調整。

| 欄位名稱 | 資料型態 | 必填性 | 欄位註釋 |
|---|---|---|---|
| `id` | `bigint` | 必填 | 主鍵，資料唯一識別碼 |
| `batch_id` | `FK` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `employee_id` | `FK` | 必填 | 員工外鍵 |
| `status_code` | `integer` | 必填 | 流程或資料狀態代碼 |
| `granted_by` | `FK` | 必填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `granted_at` | `datetime` | 選填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `cancelled_by` | `FK` | 選填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `cancelled_at` | `datetime` | 選填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `cancel_reason` | `text` | 選填 | 欄位已確認；代碼值或額外約束未在定案節點明定 |
| `created_at` | `datetime` | 必填 | 建立時間 |
| `updated_at` | `datetime` | 必填 | 最後修改時間 |

逐員工處理，個別失敗可單獨重試；公司直接核發，員工不可互贈；生效前可撤銷，已使用歷史不可抹除；到期或離職後不可使用但資料保留。




