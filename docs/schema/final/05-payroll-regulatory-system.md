# Payroll、政府法規、系統管理

## `payroll_pending_items` ✅

保存尚未進入薪資週期、但未來必須處理的項目，例如補休到期轉薪、後續補發或扣回。應能追溯 source_type/source_id、員工、金額／分鐘、適用薪資期間與處理狀態。

## `payroll_settlements` ✅

員工薪資週期結算與鎖定。定案：

1. 薪資採事後結算。
2. 結算後不可直接異動。
3. 更正採後續補發／扣回。
4. 結算保存薪資設定、事件資料與法規版本依據。

## `payroll_regulatory_snapshots` 🟡（暫定名稱）

概念已明確同意：每次結算保存實際使用的勞保、健保、勞退、職災、所得稅版本；但要獨立成表或直接放 settlement 關聯欄位尚未最終確認。

## `company_regulatory_settings` ✅

| 欄位 | 型態 | 必要 |
|---|---|---:|
| id | BIGINT | 是 |
| company_id | BIGINT | 是 |
| occupational_industry_code | VARCHAR(30) | 是 |
| insurance_unit_type_code | VARCHAR(30) | 是 |
| effective_from | DATE | 是 |
| effective_to | DATE | 否 |
| created_by | BIGINT | 是 |
| created_at | DATETIME | 是 |

只保存公司的職災行業別與投保單位類別選擇，不複製政府當期費率。

## `regulatory_dataset_versions` ✅

| 欄位 | 型態 | 必要 |
|---|---|---:|
| id | BIGINT | 是 |
| dataset_code | INT | 是 |
| version_code | VARCHAR(30) | 是 |
| effective_from | DATE | 是 |
| effective_to | DATE | 否 |
| government_resource_id | VARCHAR(150) | 否 |
| source_modified_at | DATETIME | 否 |
| synced_at | DATETIME | 是 |
| checksum | VARCHAR(128) | 是 |
| record_count | INT | 否 |
| raw_format_code | INT | 是 |
| raw_data | LONGTEXT | 是 |
| created_at | DATETIME | 是 |

建议唯一约束：`UNIQUE(dataset_code, version_code)`。version_code 統一西元，例如 2026-01。原始資料支援 JSON、CSV、XML，不使用 JSON 欄位限制原始格式。

## `regulatory_records` ✅

| 欄位 | 型態 | 必要 |
|---|---|---:|
| id | BIGINT | 是 |
| dataset_version_id | BIGINT | 是 |
| record_key | VARCHAR(150) | 是 |
| code | VARCHAR(100) | 否 |
| name | VARCHAR(250) | 否 |
| range_from/range_to | DECIMAL(18,4) | 否 |
| amount | DECIMAL(18,4) | 否 |
| rate | DECIMAL(18,8) | 否 |
| data | JSON | 是 |
| sort_order | INT | 否 |
| created_at | DATETIME | 是 |

建议唯一約束：`UNIQUE(dataset_version_id, record_key)`。所得稅特殊結構先放 data，實際串接證明不適合時再拆專表。

## `regulatory_sync_logs` ✅

| 欄位 | 型態 | 必要 |
|---|---|---:|
| id | BIGINT | 是 |
| dataset_code | INT | 是 |
| trigger_type_code | INT | 是 |
| started_at | DATETIME | 是 |
| finished_at | DATETIME | 否 |
| status_code | INT | 是 |
| dataset_version_id | BIGINT | 否 |
| government_resource_id | VARCHAR(150) | 否 |
| records_received | INT | 否 |
| error_message | TEXT | 否 |
| created_at | DATETIME | 是 |

trigger：1=自動排程、2=人工；status：1=執行中、2=更新成功、3=失敗、4=無異動。同步失敗繼續使用既有有效版本。

## 政府資料類型

1. 勞保投保薪資級距
2. 勞就保保費分擔
3. 健保投保金額級距
4. 勞退月提繳級距
5. 職災投保薪資級距
6. 職災行業別與費率
7. 薪資所得扣繳稅額表

## `audit_logs` 🟡（暫定名稱）

稽核模組已同意，但表名與欄位未最後定案。責任必須涵蓋操作者、時間、操作類型、目標資料、異動前後、登入登出、審核、核發、撤銷及權限異動。

建議欄位只能視為下一輪討論起點：id、company_id、actor_id、action_code、target_type、target_id、before_data、after_data、ip_address、device_identifier、occurred_at。
