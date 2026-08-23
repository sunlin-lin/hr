# 法規同步、系統管理與舊稿稽核

## 法規同步定案

正式四表：`company_regulatory_settings`、`regulatory_dataset_versions`、`regulatory_records`、`regulatory_sync_logs`。逐欄定義以 `docs/schema/README.md` 的法規段落為準。

核心規則：抓取／解析由程式碼管理；`version_code` 統一西元；`effective_from` 必填、`effective_to` 可空；原始內容用 `raw_format_code + LONGTEXT raw_data`；同步失敗沿用舊版；Payroll 依各法規適用基準日選版並在結算鎖定。

約束：`UNIQUE(dataset_code, version_code)`；`UNIQUE(dataset_version_id, record_key)`；有效期間不得形成無法判定的重疊。

明確淘汰：`regulatory_sources`、`regulatory_datasets`、固定 API URL、Version+Revision、政府撤回、人工核准、複雜狀態工作流、`is_current`。

## 系統管理

角色權限正式三表：`roles`、`permissions`、`role_permissions`。權限以 `permissions.parent_id` 建階層，角色與權限多對多，不預先寫死老闆、人資、主管或功能清單。

稽核日誌已確認需要記錄操作、異動、操作者與時間；正式表名及完整欄位待恢復。

## 舊稿衝突

1. 根索引的 `employee_departments`、`employee_positions` 已被後續對話更正，不得採用。
2. 排班恢復稿多個名稱明載為「建議暫稱」，只能證明概念存在。
3. 法規早期六表與 Revision 版本已被後續四表方案推翻。
4. 請假曾同時出現「不同假別不可混用」與較晚的「一張申請可有多假別明細」。較晚 Schema 採 `leave_request_details + allocations`；實作 UI 前仍建議再次確認是否允許同單混合。

## 開發使用規則

- A 級逐欄內容可作規格基礎。
- B 級只能作模組／規則基礎，不得直接產生 migration。
- 暫稱與 C 級內容不得進入正式 DB 命名。
- 衝突時以較晚、明確的確認節點為準。

