# 加班、補休、請假、公司贈與

## `overtime_requests` ✅

加班申請主單。核心方向：employee_id、employee_schedule_id、start_at、end_at、requested_minutes、reason、status、submitted_at。支援跨日；假日加班資格依實際班表。

## `overtime_approvals` ✅

審核、拒絕、撤回歷史。核心方向：overtime_request_id、action_code、action_by、action_at、reason。拒絕後可重新申請，舊歷史保留。

## `overtime_compensations` ✅

核准加班的最終補償方式。定案：一筆加班只能選 1=加班費或 2=補休，不可拆分混用。打卡超過申請時段不自動增加認列時間。

## `compensatory_leave_credits` ✅

每一批補休額度。核心方向：employee_id、source_type_code、source_id、earned_minutes、used_minutes、reserved_minutes、remaining_minutes、earned_at、expire_at、status。

來源可為加班、公司贈與、人工調整。

## `compensatory_leave_rate_snapshots` ✅

補休核發當下選定的薪資計價基準。應保存 credit/overtime 關聯、base_amount/hourly_rate、calculation_snapshot、selected_by、selected_at。後續調薪不影響此 Snapshot。

## `compensatory_leave_transactions` ✅

補休帳本，記錄取得、預約、使用、返還、到期轉薪、撤銷及重新核發。不可直接改掉原始額度歷史。

## `compensatory_leave_allocations` ✅

請假或補休使用實際扣到哪些 credit 批次。定案：最早到期優先、可部分使用、待審先凍結、取消原路返還。

## `leave_types` ✅

假別主檔。區分法定假、特休、福利假、補休等；是否有薪、是否需額度、是否需審核、是否需文件等由假別與規則決定。

## `leave_type_rules` ✅

假別法規／公司規則及生效期間。可承載額度、適用資格、薪資、文件與有效期間規則；不可使用 DB ENUM 鎖死。

## `leave_entitlements` ✅

員工實際取得的假額度批次。核心方向：employee_id、leave_type_id、entitled_minutes、used/reserved/remaining minutes、effective_from/to、source_type/source_id、status。

## `leave_balances` ✅

員工某假別的目前餘額快取／彙總；真實歷史仍以 entitlement、transaction、allocation 為準。

## `leave_balance_transactions` ✅

假額度取得、凍結、使用、返還、到期、離職失效等完整帳本。

## `leave_requests` ✅

請假申請主單。核心方向：employee_id、request_no、status、reason、applied_at、審核完成時間等。

## `leave_request_details` ✅

請假日期、起訖時間、分鐘數、假別等明細。原對話對「一張單能否混合假別」有衝突；最安全結構是主單可有多筆 detail，但每筆 detail 僅一個 leave_type。是否同單允許不同假別仍列待確認。

## `leave_request_approvals` ✅

請假審核、拒絕、取消歷史，不覆蓋原審核結果。

## `leave_request_allocations` ✅

每筆請假明細扣到哪些 entitlement／補休 credit 批次；取消時依 allocation 原路返還。

## `leave_request_documents` ✅

診斷證明、死亡證明、親屬關係等文件與驗證紀錄。方向包含 leave_request_id、document_type_code、file_id、verified_at/by。

## `leave_events` ✅

結婚、喪親、生產、流產、懷孕、配偶分娩、職災等特殊事件來源，用於判斷特殊假資格。

## `company_leave_grant_batches` ✅

公司批次核發假別。單一批次不可混用假別、pay type、額度或有效期間；批次逐員工處理，不整批 rollback。

## `company_leave_grants` ✅

批次內每位員工的核發結果。員工之間不可轉贈；granted_minutes > 0；effective_from/to 必填；失敗員工可單獨重試；生效前可撤銷；已使用歷史不可抹除。

## 定案規則

- 到期或離職後額度不可再用，但資料保留。
- 公司贈與 pay_type 僅 1=有薪、2=無薪。
- 補休到期當日仍可使用；到期剩餘轉薪。
- 已進薪資結算者不可回頭異動。
