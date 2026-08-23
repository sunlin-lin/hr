# 加班、補休、請假與公司贈與假

## 加班

### `overtime_requests`

**註釋：** 員工加班申請，保存日期、起訖、分鐘、原因、班表關聯與流程狀態。

### `overtime_approvals`

**註釋：** 加班核准、拒絕、撤回等審核歷史。

### `overtime_compensations`

**註釋：** 已核准加班最終補償方式。正式規則：一筆加班不可拆為部分加班費＋部分補休；`1=加班費`、`2=補休`。實際打卡超過申請時段不自動擴增認列。

## 補休

### `compensatory_leave_credits`

**註釋：** 每筆來源產生的補休額度批次。

核心欄位：`id`、`company_id`、`employee_id`、`employment_id`、`source_type_code`（1加班、2公司贈與、3人工調整）、`source_id/source_overtime_id`、`pay_type_code`、`credited/earned_minutes`、`effective_from/earned_at`、`effective_to/expire_at`、`status_code`、時間欄位。

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

`id bigint/uuid PK`、`employee_id FK`、`leave_type_id FK`、`source_type_code integer`（1法定取得、2公司贈與、3遞延、4人工調整）、`source_id nullable`、`pay_type_code integer`（1有薪、2無薪）、`entitled_minutes integer`、`effective_from date`、`effective_to date`、`status_code integer`、`created_at datetime`、`updated_at datetime`。

### `leave_balances` / `leave_balance_transactions`

前者為當前餘額彙總／快取；後者為取得、使用、返還、調整及到期的完整帳本。原始 `entitled_minutes` 不因使用而 UPDATE。

### `leave_requests`

`id bigint/uuid PK`、`employee_id FK`、`request_no varchar(30)`、`status_code integer`、`reason text nullable`、送出／核准／拒絕／撤銷人員與時間、拒絕／撤銷原因、`created_at datetime`、`updated_at datetime`。

### `leave_request_details`

`id PK`、`leave_request_id FK`、`leave_type_id FK`、`leave_date date`、`start_time time nullable`、`end_time time nullable`、`requested_minutes integer`、`reason text nullable`、時間欄位。一張申請可跨日期，亦可用多明細表達不同假別。

### `leave_request_allocations`

`id PK`、`leave_request_detail_id FK`、`entitlement_type_code integer`、`entitlement_id FK`、`allocated_minutes integer`、`created_at datetime`。記錄每次實際扣用哪一批一般假額度或補休額度。

### `leave_request_approvals` / `leave_request_documents` / `leave_events`

分別保存多層審核歷史、診斷／死亡／親屬等證明附件、以及結婚、死亡、生產、流產、職災等特殊事件來源。

## 公司贈與假

### `company_leave_grant_batches`

`id bigint PK`、`batch_no varchar(30)`、`name varchar(100)`、`leave_type_id FK`、`pay_type_code integer`、`granted_minutes integer`、`effective_from date`、`effective_to date`、`reason text`、`created_by FK`、`created_at datetime`、`updated_at datetime`。

同批不得混用假別、薪資類型、額度或有效期；`granted_minutes > 0`；起訖日必填。

### `company_leave_grants`

`id bigint PK`、`batch_id FK`、`employee_id FK`、`status_code integer`、`granted_by FK`、`granted_at datetime nullable`、`cancelled_by FK nullable`、`cancelled_at datetime nullable`、`cancel_reason text nullable`、`created_at datetime`、`updated_at datetime`。

逐員工處理，個別失敗可單獨重試；公司直接核發，員工不可互贈；生效前可撤銷，已使用歷史不可抹除；到期或離職後不可使用但資料保留。


