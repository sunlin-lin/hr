# 薪資設定與人事成本

> 本模組多數責任已同意，但原對話沒有留下全部逐欄最終確認；因此不得把下列欄位方向當成已完成 Migration 規格。

## `payroll_settings` 🟡

公司計薪週期、結算起訖、發薪日規則。已討論 company_id、payroll_frequency_code、payroll_start_day、payroll_end_day、payday_type_code、is_active；完整限制待定。

## `salary_items` 🟡

薪資項目主檔，涵蓋本薪、津貼、獎金、加班費、請假扣款、補休到期轉薪、其他加項與扣項。

已討論欄位方向：id、company_id、code、item_name、type_code、calculation_type_code、is_taxable、is_insurable、is_company_cost、is_active、timestamps。

定案原則：經常性／非經常性、應稅／免稅、公司成本等性質屬於薪資項目，不屬於 employees。

## `employee_salary_settings` 🟡

員工薪資設定主體。應關聯 employee 或 employment，保存生效期間，不直接寫回員工主檔。

## `employee_salary_histories` 🟡

調薪歷史。已同意任何調薪必須新增生效紀錄，不覆蓋歷史；可保存 base_salary、effective_date、reason_code 等。

## `employee_salary_items` 🟡

員工適用哪些薪資項目、金額／計算方式及生效期間。與 `salary_items` 分離。

## `payroll_periods` 🟡

公司薪資期間。已討論 company_id、period_code、start_date、end_date、pay_date、status；完整結算狀態仍需定義。

## `payroll_details` 🟡

員工當期薪資明細。應保存 salary_item_id、金額、加扣類型、計算來源與 Snapshot，不依賴之後會變動的設定重新計算歷史。

## `payroll_payments` 🟡

實際發薪紀錄，包含 payroll/employee、bank_account_id、payment_method_code、paid_at、payment_status_code、reference_number 等方向。

## `personnel_costs` 🟡

依員工／部門／期間彙整人事成本。是否為實體表、Snapshot 或報表彙總尚未定案。

## `personnel_cost_items` 🟡

人事成本的薪資、雇主負擔保險、勞退等構成。與 salary_items 的界線仍需最後確認。

## 已定案原則

1. 薪資設定、薪資項目、員工薪資分離。
2. 所有調薪與薪資項目套用都保存生效歷史。
3. 發薪時可加入一次性加項／扣項。
4. 事件發生時保存計算依據，薪資週期再結算。
5. 已結算資料不可直接異動，只能後續補發、扣回或更正。
