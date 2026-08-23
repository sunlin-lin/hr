# 人資系統需求原始對話紀錄 — Part 005

> 來源：ChatGPT 對話「整理人資系統需求」  
> 時區：Asia/Taipei  
> 範圍：2026/08/22 21:19:55 ～ 2026/08/22 21:37:13  
> 說明：保留原始對話順序，未去除重複、舊版或後續已修正內容。

---

## 2026/08/22 21:19:55

### 使用者

富項應該不會放在這吧?

### 助理

對，如果你說的是 **「薪資項目」**，我同意你這個疑問。

如果我們把 `employee_salary_items` 當成「員工薪資設定」，**不應該把所有薪資項目都塞在這裡**。

更精確的分法應該是：

```text
salary_items
    ↓
定義「公司有哪些薪資項目」

employee_salary_items
    ↓
定義「這個員工套用了哪些薪資項目」

payroll_details
    ↓
定義「這一期實際算出了哪些薪資項目」
```

例如：

### `salary_items`

公司定義：

```text
基本薪資
主管加給
交通津貼
伙食津貼
績效獎金
加班費
勞保
健保
所得稅
```

這是**薪資項目字典**。

---

### `employee_salary_items`

王小明目前的固定薪資設定：

```text
基本薪資 → 50,000
主管加給 → 5,000
交通津貼 → 2,000
```

這裡比較適合放：

```text
employee_salary_item
├── salary_item_id
├── amount
├── start_date
└── end_date
```

而不是把「實際計薪產生的項目」也放進來。

---

### `payroll_details`

到了 2026/08 計薪：

```text
基本薪資       50,000
主管加給        5,000
交通津貼        2,000
加班費          3,500
績效獎金        5,000
-------------------
應發            65,500

勞保            1,000
健保              800
所得稅            500
-------------------
實發            63,200
```

這些才是**實際薪資項目明細**。

所以你的直覺是對的：

> **「薪資項目」本身應該獨立成 `salary_items`，而不是直接塞進員工薪資資料。**

而且我再往前想一步：

**「員工薪資設定」甚至可能不需要 `salary_item_id + amount` 這麼簡單。**

因為基本薪資、津貼、獎金、加班費的計算方式完全不同。

所以我建議下一步我們先只討論：

### `salary_items` 到底要有哪些欄位

先把「薪資項目定義」設計好，再往下設計員工薪資設定與計薪。

---

## 2026/08/22 21:20:49

### 使用者

你覺得還缺什麼

### 助理

有。如果從「**可以長期運作的台灣 HR SaaS**」來看，目前薪資模型還缺幾個重要概念。

我會先不急著定 Schema，而是先把需求層次補齊。

## 目前薪資應該拆成 5 層

```text
薪資項目定義
    ↓
員工薪資設定
    ↓
薪資異動歷史
    ↓
計薪期間
    ↓
薪資結果 / 薪資明細
```

---

## 1. 薪資項目 `salary_items` ✅

這個我們已經談到。

例如：

```text
基本薪資
職務加給
主管加給
交通津貼
伙食津貼
全勤獎金
績效獎金
加班費
勞保
健保
所得稅
```

但這張表還需要考慮：

- 項目名稱
- 項目代碼
- 應發 / 扣款
- 是否課稅
- 是否列入投保薪資
- 計算方式
- 是否固定
- 是否允許手動輸入

這些之後再細定。

---

# 2. 員工薪資設定

例如：

```text
王小明

基本薪資       50,000
主管加給        5,000
交通津貼        2,000
```

這是「**平常的薪資設定**」。

但這裡有一個很重要的問題：

### 薪資設定不一定是固定金額

例如：

```text
交通津貼 = 2,000
```

很簡單。

但：

```text
主管加給 = 基本薪資 × 10%
```

又或者：

```text
伙食津貼 = 每工作日 × 100
```

所以員工薪資設定至少要能表達：

```text
固定金額
比例
數量 × 單價
公式
```

這個我們後面一定要討論。

---

# 3. 薪資異動歷史 ⚠️

這個我認為一定要有。

例如：

```text
基本薪資

2025/01/01 → 40,000
2025/07/01 → 43,000
2026/01/01 → 46,000
```

不能只修改：

```text
amount = 46000
```

否則以前的薪資資料會消失。

所以：

> **薪資設定本身就必須具有有效期間。**

也就是我們前面提到的：

```text
start_date
end_date
```

---

# 4. 計薪期間

這個非常容易被忽略。

例如公司可能：

```text
每月 1 日～月底
```

也可能：

```text
每月 26 日～次月 25 日
```

甚至不同公司不同。

所以不能直接假設：

```text
2026/08 = 8/1 ~ 8/31
```

應該有：

```text
payroll_periods
```

概念：

```text
2026/08 薪資

計薪開始：2026/07/26
計薪結束：2026/08/25
發薪日：2026/08/31
```

---

# 5. 薪資結果

這是另外一個世界。

例如：

```text
2026/08
王小明

基本薪資       50,000
主管加給        5,000
加班費          3,200
-------------------
應發            58,200

勞保            1,000
健保              800
所得稅            500
-------------------
實發            55,900
```

這些**不能回頭去讀目前的員工薪資設定來代替**。

因為：

> 薪資結果是歷史帳。

所以一定要保存。

---

# 6. 還缺「薪資週期 / 發薪制度」

這個 SaaS 很重要。

公司 A：

```text
月薪
每月 5 日發薪
```

公司 B：

```text
月薪
每月 10 日發薪
```

公司 C：

```text
半月薪
每月 15 日、30 日各發一次
```

所以：

```text
payroll_frequency
```

可能需要：

```text
月薪
週薪
雙週
半月
```

而且「計薪期間」與「發薪日」要分開。

---

# 7. 缺「薪資銀行帳戶」

這個我認為**一定會需要**。

員工薪資最後要匯款：

```text
銀行
分行
帳號
戶名
```

而且這也是敏感資料。

例如：

```text
bank_code
branch_code
account_number_encrypted
account_number_hash
account_name
```

甚至一個員工未來可能有：

```text
主要帳戶
第二帳戶
```

所以我不建議直接塞進 `employees`。

應該獨立：

```text
employee_salary_bank_accounts
```

---

# 8. 缺「薪資發放方式」

例如：

```text
銀行轉帳
現金
支票
```

這也是薪資設定的一部分。

而且未來可能需要記錄：

```text
薪資已發
薪資待發
部分發放
發放失敗
```

這應該屬於 Payroll 結果，而不是員工基本資料。

---

# 9. 缺「薪資鎖定」

這個我認為 SaaS **一定要設計**。

例如：

```text
2026/08 薪資

草稿
 ↓
計算
 ↓
審核
 ↓
核准
 ↓
發薪
 ↓
結算
```

一旦：

```text
已發薪
```

就不能讓管理員直接修改：

```text
基本薪資
加班費
勞保
所得稅
```

否則歷史帳會被破壞。

所以 Payroll 必須有自己的生命週期。

---

# 10. 還有「手動調整」

實務上一定會遇到：

```text
這個員工這個月多發 3,000
```

例如：

- 臨時獎金
- 補發
- 扣回
- 前月差額
- 特殊津貼
- 人工作業調整

不能要求所有東西都進公式。

所以 Payroll 必須允許：

```text
手動增加
手動扣除
```

並記錄：

```text
原因
操作人
時間
```

---

# 11. 最後一個很重要：薪資計算不要跟打卡綁死

你前面的人事系統還有：

```text
打卡
請假
```

未來薪資會吃這些資料：

```text
打卡
 ↓
工時
 ↓
加班
 ↓
請假
 ↓
應扣薪
 ↓
Payroll
```

但：

> **Payroll 不應該直接負責管理打卡或請假。**

它只是讀取結果。

這樣模組邊界會非常漂亮。

---

# 我目前認為薪資至少要有這些核心概念

```text
salary_items
        │
        ▼
employee_salary_settings
        │
        ▼
salary_history
        │
        ▼
payroll_periods
        │
        ▼
payrolls
        │
        ▼
payroll_details
        │
        ├── 薪資項目
        ├── 加班
        ├── 請假扣薪
        ├── 獎金
        ├── 扣款
        └── 手動調整

employee_salary_bank_accounts
```

### 但我會建議現在先不要一次把 Schema 全定掉。

下一步最值得先決定的是：

**「薪資項目 `salary_items` 到底要怎麼定義？」**

因為它會決定後面：

> 固定薪資、津貼、獎金、加班費、扣款、勞健保、稅額、公式計算

到底怎麼串。

如果 `salary_items` 設計得好，後面的 Payroll 會簡單很多。

---

## 2026/08/22 21:23:45

### 使用者

3.同意
4.同意
5.同意
6.同意
7.同意
8.同意
9.同意
10.同意
11.同意

### 助理

好，這些我就視為**已確認需求**，後續薪資 Schema 會以這些規則為基準。

目前薪資模組已確認：

| # | 項目 | 狀態 |
|---|---|---|
| 1 | 薪資項目 `salary_items` | 🟡 尚未正式定義 |
| 2 | 員工薪資設定 | 🟡 尚未正式定義 |
| 3 | 薪資異動歷史 | ✅ 確定 |
| 4 | 計薪期間 | ✅ 確定 |
| 5 | 薪資結果 | ✅ 確定 |
| 6 | 薪資週期 / 發薪制度 | ✅ 確定 |
| 7 | 薪資銀行帳戶 | ✅ 確定 |
| 8 | 薪資發放方式 | ✅ 確定 |
| 9 | 薪資鎖定 / 審核 / 結算 | ✅ 確定 |
| 10 | 手動薪資調整 | ✅ 確定 |
| 11 | Payroll 與打卡、請假分離 | ✅ 確定 |

### 現在的薪資資料關係

```text
company
   │
   ├── salary_items
   │
   ├── payroll_settings
   │
   └── employee
          │
          ├── employee_salary_settings
          │
          └── employee_salary_bank_accounts

salary settings
       │
       ▼
payroll_periods
       │
       ▼
payrolls
       │
       ▼
payroll_details
       │
       ├── salary items
       ├── overtime
       ├── leave deduction
       ├── bonus
       ├── deduction
       └── manual adjustment
```

另外有一個重要原則也一起確定：

> **薪資設定是「現在應該怎麼算」，Payroll 是「當時實際算出什麼」。**

因此即使之後員工調薪、薪資項目修改，**已結算的歷史 Payroll 絕對不能跟著變動**。

---

接下來我建議正式開始設計 **`salary_items` Schema**。

這張是整個薪資系統的核心，我們可以先只討論這一張，不急著往後面做。

---

## 2026/08/22 21:24:32

### 使用者

1.可以先擬一份預設的
2.這是什麼，在說明一次

### 助理

可以，兩件事分開處理。

## 1. 先擬一份系統預設 `salary_items`

我建議**不要一開始塞太細**，先提供一批台灣企業普遍會用到的薪資項目，企業之後可以自行新增。

### 預設薪資項目

| code | 名稱 | 類型 | 說明 |
|---|---|---|---|
| `BASE` | 基本薪資 | 應發 | 員工主要固定薪資 |
| `POSITION` | 職務加給 | 應發 | 因擔任特定職務所給付 |
| `MANAGEMENT` | 主管加給 | 應發 | 主管職務津貼 |
| `SKILL` | 技能加給 | 應發 | 特殊技能或專業能力加給 |
| `TRANSPORT` | 交通津貼 | 應發 | 交通相關津貼 |
| `MEAL` | 伙食津貼 | 應發 | 伙食相關津貼 |
| `ATTENDANCE` | 全勤獎金 | 應發 | 全勤相關獎勵 |
| `PERFORMANCE` | 績效獎金 | 應發 | 績效相關獎金 |
| `BONUS` | 其他獎金 | 應發 | 其他獎金 |
| `OVERTIME` | 加班費 | 應發 | 加班產生的薪資 |
| `LEAVE_PAY` | 假日／休假薪資 | 應發 | 依相關規則產生 |
| `OTHER_EARNING` | 其他應發 | 應發 | 其他收入 |
| `LABOR_INSURANCE` | 勞保費 | 扣款 | 員工負擔部分 |
| `HEALTH_INSURANCE` | 健保費 | 扣款 | 員工負擔部分 |
| `INCOME_TAX` | 所得稅 | 扣款 | 薪資所得扣繳 |
| `LEAVE_DEDUCTION` | 請假扣款 | 扣款 | 因請假產生的扣薪 |
| `OTHER_DEDUCTION` | 其他扣款 | 扣款 | 其他員工負擔扣款 |

這些是**系統預設項目**，不是限制。

例如公司可以再建立：

```text
交通補助
手機津貼
證照津貼
留任獎金
專案獎金
生日禮金
```

而且我會讓：

```text
company_id = NULL
is_system = true
```

代表系統預設項目。

公司自行建立的：

```text
company_id = 該公司
is_system = false
```

---

# 2. 「員工薪資設定」到底是什麼？

這個要特別講清楚。

`salary_items` 是：

> **「薪資系統有哪些薪資項目？」**

例如：

```text
基本薪資
主管加給
交通津貼
績效獎金
```

但是它不知道**王小明到底有沒有主管加給**。

所以才需要「員工薪資設定」。

---

### 舉例

公司有：

```text
salary_items

基本薪資
主管加給
交通津貼
績效獎金
```

但員工王小明：

```text
基本薪資      50,000
主管加給       5,000
交通津貼       2,000
```

李小華：

```text
基本薪資      40,000
交通津貼       2,000
```

因此：

### `salary_items`

是「**項目定義**」。

### `employee_salary_settings`

是「**這個員工套用了哪些項目，以及怎麼算**」。

---

## 為什麼我現在想改名？

前面我叫它：

```text
employee_salary_items
```

我現在比較建議叫：

```text
employee_salary_settings
```

因為這張表真正的意思不是「薪資項目」，而是：

> **員工薪資計算設定。**

例如：

```text
王小明
│
├── 基本薪資
│   └── 固定 50,000
│
├── 主管加給
│   └── 固定 5,000
│
└── 交通津貼
    └── 固定 2,000
```

而未來可能變成：

```text
基本薪資
→ 固定金額

交通津貼
→ 每工作日 100 元

績效獎金
→ 基本薪資 × 10%

加班費
→ 依加班時數計算
```

所以它不是單純存：

```text
salary = 50000
```

而是存：

> **「這個員工的這個薪資項目，要怎麼計算。」**

---

## 所以目前先把兩張表的定位定清楚

```text
salary_items
```

### 公司／系統的「薪資項目目錄」

回答：

> **有哪些薪資項目？**

↓

```text
employee_salary_settings
```

### 個別員工的「薪資計算設定」

回答：

> **這個員工有哪些薪資項目？每個項目怎麼算？何時生效？**

↓

```text
payroll
```

### 實際某一期的「計薪結果」

回答：

> **這個員工 2026 年 8 月實際拿多少？**

這三層分開後，整個薪資模型就會非常清楚。

---

## 2026/08/22 21:26:10

### 使用者

可以，例外在發放薪資時，要可以臨時增加發放項目

### 助理

可以，而且我認為**這個需求一定要保留**。

但我會把它定義成：

> **薪資發放時的臨時薪資項目，不修改員工的長期薪資設定。**

例如王小明平常：

```text
基本薪資      50,000
主管加給       5,000
交通津貼       2,000
```

8 月發薪時，公司臨時決定：

```text
專案獎金       10,000
```

這個 `10,000` **不能寫進 `employee_salary_settings`**，因為它不是長期薪資設定。

應該直接進入當期 Payroll：

```text
2026/08 Payroll
│
├── 基本薪資       50,000
├── 主管加給        5,000
├── 交通津貼        2,000
└── 專案獎金       10,000 ← 臨時增加
```

---

## 所以薪資項目會有三種來源

### ① 員工固定薪資設定

來自：

```text
employee_salary_settings
```

例如：

```text
基本薪資 50,000
主管加給 5,000
```

---

### ② 系統自動計算

例如：

```text
加班費
請假扣款
勞保
健保
所得稅
```

由 Payroll 計算產生。

---

### ③ 發薪時臨時增加

例如：

```text
專案獎金       10,000
生日禮金        2,000
補發薪資        3,000
前月差額        1,500
```

直接加入**該期薪資明細**。

---

# 這也代表 `payroll_details` 要設計得比較彈性

它不能單純只有：

```text
salary_item_id
amount
```

還需要知道：

```text
來源
```

例如：

```text
source_type_code
```

可以先規劃：

```text
1 = 員工薪資設定
2 = 系統計算
3 = 臨時新增
4 = 人工調整
```

而且臨時項目最好可以：

```text
salary_item_id = NULL
```

因為有些臨時項目甚至不值得建立成公司的永久薪資項目。

例如主管臨時輸入：

> 「客戶專案獎勵」

```text
項目名稱：客戶專案獎勵
金額：10,000
```

不一定要先去 `salary_items` 建一個正式項目。

---

## 我會建議再加「備註 / 原因」

臨時薪資尤其重要：

```text
item_name = 客戶專案獎勵
amount = 10000
source_type_code = 3
reason = XX專案提前完成獎勵
```

並記錄：

```text
created_by
created_at
```

這樣日後查薪資：

> 為什麼這個月多了 10,000？

可以直接追到：

```text
誰加的
什麼時候加的
加了什麼
多少錢
為什麼
```

---

### 所以目前薪資模型又確定一條重要規則

**員工薪資設定 ≠ 當期薪資明細。**

```text
employee_salary_settings
        │
        │ 平常設定
        ▼
     Payroll
        │
        ├── 固定薪資
        ├── 系統計算
        ├── 臨時增加
        └── 人工調整
        │
        ▼
payroll_details
```

而且**一旦 Payroll 核准／結算，當期明細就成為歷史帳，不再跟著員工目前的薪資設定變動。**

這個我建議正式記錄到薪資模組規則裡。

---

## 2026/08/22 21:28:55

### 使用者

我看不懂

### 助理

簡單講，我剛才講太複雜了。

你只要先記住 **3 張東西** 就好。

---

## ① `salary_items`：公司有哪些薪資項目

這是「目錄」。

例如公司預設有：

```text
基本薪資
主管加給
交通津貼
伙食津貼
績效獎金
加班費
勞保
健保
所得稅
```

它只是在說：

> **公司允許有哪些薪資項目。**

---

## ② `employee_salary_settings`：這個員工平常怎麼算薪水

例如王小明：

```text
基本薪資    50,000
主管加給     5,000
交通津貼     2,000
```

這表示：

> **王小明平常每個月有這些薪資設定。**

如果 2026/09 調薪：

```text
基本薪資
2026/01/01 → 50,000
2026/09/01 → 55,000
```

就保留歷史。

---

## ③ `payroll_details`：這個月實際發了什麼

假設 2026 年 8 月：

```text
基本薪資      50,000
主管加給       5,000
交通津貼       2,000
```

這些是正常從員工薪資設定帶進來的。

但是這個月公司突然要多發：

> 「專案獎金 10,000」

你希望可以**直接在發薪時增加**。

所以 Payroll 裡面變成：

```text
2026 年 8 月薪資

基本薪資       50,000
主管加給        5,000
交通津貼        2,000
專案獎金       10,000  ← 臨時增加
----------------------
應發            67,000
```

這個「專案獎金 10,000」：

**不會跑去修改王小明的平常薪資設定。**

所以 9 月薪資還是：

```text
基本薪資      50,000
主管加給       5,000
交通津貼       2,000
```

而不是變成：

```text
基本薪資      50,000
主管加給       5,000
交通津貼       2,000
專案獎金      10,000  ← 不會留下來
```

---

# 最簡單的概念

```text
salary_items
「有哪些薪資項目？」

        ↓

employee_salary_settings
「這個員工平常有哪些薪資？」

        ↓

payroll_details
「這個月實際發了哪些錢？」

                    ↑
                    │
              可以臨時增加
              專案獎金 10,000
```

所以你的需求完全沒問題：

> **發薪時可以臨時增加薪資項目，而且只影響這一期，不修改員工原本的薪資設定。**

這個需求我會直接納入薪資 Schema。

---

## 2026/08/22 21:31:48

### 使用者

把Schema  給我看

### 助理

可以。現在我們先把**薪資核心 Schema**整理成一版，而且先不碰程式架構。

我會按照我們剛剛確定的規則來設計：

- 不使用 `enum`
- 固定選項使用 `integer/string code`
- 薪資設定與實際發薪結果分離
- 薪資要保留歷史
- 發薪時可以臨時增加項目
- 已結算的薪資不能因為後來調薪而改變
- 敏感資料使用 `binary/varbinary` 加密

---

# 1. `salary_items`

### 說明

**薪資項目定義表。**

用來定義公司可以使用的薪資項目，例如基本薪資、主管加給、交通津貼、加班費、勞保、健保等。

系統可以預先提供一批預設項目，公司也可以自行建立薪資項目。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資項目唯一識別碼，主鍵 |
| `company_id` | `uuid` | 所屬公司 ID；`NULL` 表示系統預設項目 |
| `code` | `string` | 薪資項目代碼 |
| `name` | `string` | 薪資項目名稱 |
| `type_code` | `integer` | 項目類型：`1=應發`、`2=扣款` |
| `calculation_type_code` | `integer` | 計算方式代碼，後續定義 |
| `is_taxable` | `boolean` | 是否計入所得稅計算 |
| `is_insurable` | `boolean` | 是否計入投保薪資計算 |
| `is_active` | `boolean` | 是否啟用 |
| `description` | `string` | 項目說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 2. `employee_salary_settings`

### 說明

**員工薪資設定表。**

記錄某個員工目前有哪些薪資項目，以及該薪資項目的計算設定。

例如：

```text
王小明

基本薪資    50,000
主管加給     5,000
交通津貼     2,000
```

薪資調整時，不覆蓋歷史資料，而是建立新的生效期間。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工薪資設定唯一識別碼，主鍵 |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `salary_item_id` | `uuid` | 薪資項目 ID |
| `calculation_type_code` | `integer` | 該員工此項目的計算方式 |
| `amount` | `decimal` | 固定金額或計算基礎數值 |
| `start_date` | `date` | 生效日期 |
| `end_date` | `date` | 結束日期，可 NULL |
| `description` | `string` | 薪資設定說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

### 例如

```text
基本薪資
2025/01/01 → 40,000
2025/07/01 → 45,000
2026/01/01 → 50,000
```

歷史全部保留。

---

# 3. `payroll_settings`

### 說明

**公司計薪制度設定。**

定義公司多久計薪一次、計薪週期以及發薪規則。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 計薪設定唯一識別碼 |
| `company_id` | `uuid` | 所屬公司 ID |
| `payroll_frequency_code` | `integer` | 計薪週期：例如 `1=月薪`、`2=半月`、`3=雙週`、`4=週薪` |
| `payroll_start_day` | `integer` | 計薪週期起始日 |
| `payroll_end_day` | `integer` | 計薪週期結束日 |
| `payday_type_code` | `integer` | 發薪日規則 |
| `payday` | `integer` | 發薪日 |
| `description` | `string` | 設定說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

例如：

```text
計薪期間：
每月 26 日～次月 25 日

發薪日：
每月 5 日
```

---

# 4. `payroll_periods`

### 說明

**實際的計薪期間。**

`payroll_settings` 是規則，而 `payroll_periods` 是實際產生的期間。

例如：

```text
2026/08 薪資

開始：2026/07/26
結束：2026/08/25
發薪日：2026/09/05
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 計薪期間 ID |
| `company_id` | `uuid` | 所屬公司 ID |
| `period_code` | `string` | 計薪期間代碼，例如 `202608` |
| `start_date` | `date` | 計薪開始日期 |
| `end_date` | `date` | 計薪結束日期 |
| `pay_date` | `date` | 預定發薪日期 |
| `status` | `string` | 計薪期間狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 5. `payrolls`

### 說明

**員工在某一個計薪期間的薪資單主檔。**

一名員工一個計薪期間原則上只有一筆。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資單唯一識別碼 |
| `payroll_period_id` | `uuid` | 計薪期間 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `gross_amount` | `decimal` | 應發總額 |
| `deduction_amount` | `decimal` | 扣款總額 |
| `net_amount` | `decimal` | 實發金額 |
| `status` | `string` | 薪資單狀態 |
| `approved_by` | `uuid` | 核准人員 ID，可 NULL |
| `approved_at` | `datetime` | 核准時間，可 NULL |
| `paid_at` | `datetime` | 實際發薪時間，可 NULL |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 6. `payroll_details`

### 說明

**薪資單明細。**

真正記錄：

```text
基本薪資       50,000
主管加給        5,000
交通津貼        2,000
專案獎金       10,000
勞保            1,000
健保              800
所得稅            500
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資明細 ID |
| `payroll_id` | `uuid` | 薪資單 ID |
| `salary_item_id` | `uuid` | 薪資項目 ID，可 NULL |
| `item_name` | `string` | 當期薪資項目名稱快照 |
| `type_code` | `integer` | `1=應發`、`2=扣款` |
| `source_type_code` | `integer` | 明細來源 |
| `amount` | `decimal` | 實際金額 |
| `description` | `string` | 明細說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

### `source_type_code`

先定義：

```text
1 = 員工薪資設定
2 = 系統計算
3 = 臨時新增
4 = 人工調整
```

---

## 臨時新增就是這樣

例如 8 月突然要多發：

```text
專案獎金 10,000
```

直接建立：

```text
payroll_details

salary_item_id = NULL
item_name = "專案獎金"
type_code = 1
source_type_code = 3
amount = 10000
```

所以**不會改動**：

```text
employee_salary_settings
```

---

# 7. `employee_salary_bank_accounts`

### 說明

**員工薪資匯款帳戶。**

因為銀行帳號屬於敏感資料，所以獨立保存。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 銀行帳戶 ID |
| `employee_id` | `uuid` | 員工 ID |
| `bank_code` | `string` | 銀行代碼 |
| `branch_code` | `string` | 分行代碼 |
| `account_number_encrypted` | `binary` | 銀行帳號，加密 |
| `account_number_hash` | `binary` | 銀行帳號 Hash |
| `account_name_encrypted` | `binary` | 戶名，加密 |
| `is_primary` | `boolean` | 是否為主要薪資帳戶 |
| `status` | `string` | 帳戶狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 8. `payroll_payments`

### 說明

**薪資實際發放紀錄。**

`payrolls` 是「算出來多少」，這張是「實際怎麼發出去」。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 發薪紀錄 ID |
| `payroll_id` | `uuid` | 薪資單 ID |
| `payment_method_code` | `integer` | 發薪方式，例如 `1=銀行轉帳`、`2=現金`、`3=支票` |
| `bank_account_id` | `uuid` | 使用的銀行帳戶，可 NULL |
| `amount` | `decimal` | 實際發放金額 |
| `payment_status` | `string` | 發放狀態 |
| `paid_at` | `datetime` | 發放時間 |
| `reference_number` | `string` | 銀行交易或公司內部交易編號，可 NULL |
| `description` | `string` | 發放說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 整體關係

```text
                    salary_items
                         │
                         ▼
              employee_salary_settings
                         │
                         │
employee ──────── employment
                         │
                         │
                  payroll_periods
                         │
                         ▼
                      payrolls
                         │
                         ▼
                  payroll_details
                         │
                         ▼
                 payroll_payments
```

另外：

```text
employee
   │
   └── employee_salary_bank_accounts
```

---

## 我特別建議保留的一個設計

`payroll_details` 裡面的：

```text
salary_item_id
item_name
```

**兩個都留。**

因為 `salary_item_id` 是「它原本是哪個薪資項目」，而 `item_name` 是**當時發薪時的名稱快照**。

例如公司後來把：

```text
交通津貼
```

改名成：

```text
交通補助
```

2026 年 8 月的舊薪資單仍然應該顯示：

```text
交通津貼 2,000
```

而不是跟著現在的設定變成「交通補助」。

這就是為什麼 **Payroll 必須保存自己的歷史快照**。

---

## 2026/08/22 21:35:01

### 使用者

那人事成本適合放在這一次?

### 助理

我會建議：**要放，但不要把「人事成本」直接塞進薪資表。**

因為「薪資」和「人事成本」其實是兩個不同概念。

### 薪資

回答：

> **員工實際拿多少錢？**

例如：

```text
基本薪資       50,000
主管加給        5,000
交通津貼        2,000
所得稅          -500
實發            56,500
```

### 人事成本

回答：

> **公司為這個員工總共付出了多少成本？**

可能是：

```text
員工薪資       57,000
公司負擔勞保     2,000
公司負擔健保     3,000
勞退             3,000
其他福利         1,000
----------------------
公司人事成本    66,000
```

所以兩者不能混在一起。

---

# 我建議這一次「先納入人事成本的概念」

但**不要現在就把完整人事成本 Schema 做完**。

因為人事成本之後可能會延伸到：

- 公司負擔勞保
- 公司負擔健保
- 勞退
- 員工福利
- 獎金
- 教育訓練
- 員工保險
- 餐費
- 交通
- 招募成本
- 部門成本
- 專案成本

甚至：

```text
研發部
    人事成本 3,500,000

業務部
    人事成本 2,100,000
```

這已經開始接近**成本會計 / 管理會計**了。

---

## 我會把它分成兩層

### 第一層：薪資模組

現在做：

```text
salary_items
employee_salary_settings
payroll_periods
payrolls
payroll_details
payroll_payments
employee_salary_bank_accounts
```

負責：

> **算薪資、發薪資、保存薪資歷史。**

---

### 第二層：人事成本

後續建立：

```text
personnel_costs
```

負責：

> **公司實際承擔的員工成本。**

它可以引用 Payroll：

```text
payroll
   │
   ├── 員工應發
   ├── 員工扣款
   │
   └── 公司負擔成本
          │
          ▼
    personnel_costs
```

---

# 但有一個我認為現在就要保留的欄位

在 `salary_items` 裡面，我們現在有：

```text
is_taxable
is_insurable
```

我會再考慮增加：

```text
is_company_cost
```

意思是：

> 這個薪資項目是否屬於公司成本。

例如：

| 薪資項目 | 公司成本 |
|---|---|
| 基本薪資 | ✅ |
| 主管加給 | ✅ |
| 交通津貼 | ✅ |
| 績效獎金 | ✅ |
| 員工負擔勞保 | ❌ |
| 員工負擔健保 | ❌ |
| 所得稅 | ❌ |
| 公司負擔勞保 | ✅ |
| 公司負擔健保 | ✅ |
| 勞退 | ✅ |

不過「公司負擔勞保 / 健保 / 勞退」到底要不要也當成 `salary_items`，我反而建議**下一階段再討論**。

---

### 所以我的結論

**人事成本適合納入這套 HR SaaS，但不適合現在直接和薪資 Schema 混在一起。**

我會採：

```text
人事
  ↓
薪資
  ↓
Payroll
  ↓
人事成本
```

而且未來我們可以做到：

> **公司 → 部門 → 員工 → 月份 → 人事成本**

這對 SaaS 的管理報表會非常有價值。

目前先把**薪資本身定好**，人事成本先保留接口與概念，不要過早把它做複雜。

---

## 2026/08/22 21:36:11

### 使用者

那我們先把薪資本身決定，在幫我總結一次薪資

### 助理

可以。現在先把**「薪資本身」定下來，人事成本先不納入**。

我把前面討論過的內容整理成一版比較乾淨的規格。

# 薪資模組目前確定版

整體分成：

```text
薪資項目
    ↓
員工薪資設定
    ↓
計薪期間
    ↓
薪資單
    ↓
薪資明細
    ↓
實際發薪
```

另外員工有：

```text
員工
 └── 薪資銀行帳戶
```

---

# 1. `salary_items`

### 說明

**薪資項目定義表。**

定義系統或公司可以使用的薪資項目。

例如：

```text
基本薪資
職務加給
主管加給
技能加給
交通津貼
伙食津貼
全勤獎金
績效獎金
其他獎金
加班費
勞保費
健保費
所得稅
請假扣款
其他扣款
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資項目 ID |
| `company_id` | `uuid` | 公司 ID，NULL 表示系統預設項目 |
| `code` | `string` | 薪資項目代碼 |
| `name` | `string` | 薪資項目名稱 |
| `type_code` | `integer` | 項目類型，1=應發、2=扣款 |
| `calculation_type_code` | `integer` | 計算方式代碼 |
| `is_taxable` | `boolean` | 是否列入所得稅計算 |
| `is_insurable` | `boolean` | 是否列入投保薪資計算 |
| `is_active` | `boolean` | 是否啟用 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 2. `employee_salary_settings`

### 說明

**員工薪資設定表。**

記錄員工平常有哪些薪資項目，以及每個項目的計算設定。

例如：

```text
王小明

基本薪資    50,000
主管加給     5,000
交通津貼     2,000
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 員工薪資設定 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `salary_item_id` | `uuid` | 薪資項目 ID |
| `calculation_type_code` | `integer` | 計算方式代碼 |
| `amount` | `decimal` | 金額或計算基礎值 |
| `start_date` | `date` | 生效日期 |
| `end_date` | `date` | 結束日期，可 NULL |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

### 重點

薪資調整**不覆蓋舊資料**。

例如：

```text
基本薪資

2025/01/01 → 40,000
2025/07/01 → 45,000
2026/01/01 → 50,000
```

這樣可以完整追蹤歷史。

---

# 3. `payroll_settings`

### 說明

**公司計薪制度設定。**

定義公司多久計薪一次，以及計薪期間、發薪日等規則。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 計薪設定 ID |
| `company_id` | `uuid` | 公司 ID |
| `payroll_frequency_code` | `integer` | 計薪週期代碼 |
| `payroll_start_day` | `integer` | 計薪開始日 |
| `payroll_end_day` | `integer` | 計薪結束日 |
| `payday_type_code` | `integer` | 發薪日規則 |
| `payday` | `integer` | 發薪日 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

例如：

```text
計薪期間：每月 26 日～次月 25 日
發薪日：每月 5 日
```

---

# 4. `payroll_periods`

### 說明

**實際產生的計薪期間。**

`payroll_settings` 是「規則」。

`payroll_periods` 是「實際哪一期」。

例如：

```text
2026/08 薪資

計薪開始：2026/07/26
計薪結束：2026/08/25
發薪日：2026/09/05
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 計薪期間 ID |
| `company_id` | `uuid` | 公司 ID |
| `period_code` | `string` | 計薪期間代碼，例如 `202608` |
| `start_date` | `date` | 開始日期 |
| `end_date` | `date` | 結束日期 |
| `pay_date` | `date` | 預定發薪日 |
| `status_code` | `integer` | 計薪期間狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 5. `payrolls`

### 說明

**員工某一期的薪資單主檔。**

例如：

```text
王小明
2026/08
應發 67,000
扣款 2,300
實發 64,700
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資單 ID |
| `payroll_period_id` | `uuid` | 計薪期間 ID |
| `employee_id` | `uuid` | 員工 ID |
| `employment_id` | `uuid` | 任職紀錄 ID |
| `gross_amount` | `decimal` | 應發總額 |
| `deduction_amount` | `decimal` | 扣款總額 |
| `net_amount` | `decimal` | 實發金額 |
| `status_code` | `integer` | 薪資單狀態 |
| `approved_by` | `uuid` | 核准人員 ID |
| `approved_at` | `datetime` | 核准時間 |
| `paid_at` | `datetime` | 實際發薪時間 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 6. `payroll_details`

### 說明

**薪資單的實際明細。**

例如：

```text
基本薪資       50,000
主管加給        5,000
交通津貼        2,000
專案獎金       10,000
-------------------
應發            67,000

勞保            1,000
健保              800
所得稅            500
-------------------
扣款             2,300
```

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 薪資明細 ID |
| `payroll_id` | `uuid` | 薪資單 ID |
| `salary_item_id` | `uuid` | 薪資項目 ID，可 NULL |
| `item_name` | `string` | 當期項目名稱快照 |
| `type_code` | `integer` | 1=應發、2=扣款 |
| `source_type_code` | `integer` | 明細來源 |
| `amount` | `decimal` | 實際金額 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

### `source_type_code`

目前先定：

```text
1 = 員工薪資設定
2 = 系統計算
3 = 臨時新增
4 = 人工調整
```

---

## 臨時增加薪資

這是我們剛剛特別確認的需求。

例如 8 月：

```text
基本薪資       50,000
主管加給        5,000
交通津貼        2,000
專案獎金       10,000 ← 臨時增加
```

這個專案獎金直接進：

```text
payroll_details
```

而：

```text
employee_salary_settings
```

**完全不變。**

所以 9 月不會自動再出現這筆專案獎金。

---

# 7. `employee_salary_bank_accounts`

### 說明

**員工薪資發放銀行帳戶。**

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 銀行帳戶 ID |
| `employee_id` | `uuid` | 員工 ID |
| `bank_code` | `string` | 銀行代碼 |
| `branch_code` | `string` | 分行代碼 |
| `account_number_encrypted` | `varbinary` | 銀行帳號，加密 |
| `account_number_hash` | `varbinary` | 銀行帳號 Hash |
| `account_name_encrypted` | `varbinary` | 戶名，加密 |
| `is_primary` | `boolean` | 是否主要帳戶 |
| `status_code` | `integer` | 帳戶狀態 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |
| `deleted_at` | `datetime` | 軟刪除時間 |

---

# 8. `payroll_payments`

### 說明

**薪資實際發放紀錄。**

`payrolls` 是「算出多少」。

`payroll_payments` 是「實際怎麼發」。

### Schema

| 欄位 | 型態 | 說明 |
|---|---|---|
| `id` | `uuid` | 發薪紀錄 ID |
| `payroll_id` | `uuid` | 薪資單 ID |
| `payment_method_code` | `integer` | 發薪方式 |
| `bank_account_id` | `uuid` | 使用的銀行帳戶，可 NULL |
| `amount` | `decimal` | 實際發放金額 |
| `payment_status_code` | `integer` | 發放狀態 |
| `paid_at` | `datetime` | 發放時間 |
| `reference_number` | `string` | 交易編號 |
| `description` | `string` | 說明 |
| `created_at` | `datetime` | 建立時間 |
| `updated_at` | `datetime` | 修改時間 |

---

# 最終關係

```text
                        salary_items
                             │
                             ▼
                 employee_salary_settings
                             │
                             │
employee ─────── employment  │
   │                         │
   └── bank_accounts         │
                             ▼
                      payroll_periods
                             │
                             ▼
                          payrolls
                             │
                             ▼
                      payroll_details
                             │
                             ▼
                     payroll_payments
```

## 目前薪資模組「確定」的原則

1. **薪資項目獨立管理**
2. **員工薪資設定獨立**
3. **調薪保留歷史**
4. **計薪期間獨立**
5. **薪資結果獨立**
6. **公司可以設定自己的計薪週期與發薪規則**
7. **員工銀行帳戶獨立**
8. **支援不同發薪方式**
9. **薪資有草稿 → 審核 → 核准 → 發薪 → 結算的生命週期**
10. **發薪時可以臨時增加薪資項目**
11. **臨時項目不會修改員工長期薪資設定**
12. **Payroll 保存當時的薪資明細，不受未來薪資調整影響**
13. **打卡、請假提供資料給薪資計算，但不直接混入 Payroll 的管理責任**
14. **本階段不納入人事成本**
15. **不使用 ENUM，代碼類欄位使用 `integer` / `string code`**
16. **銀行帳號等敏感資料使用 `varbinary` 加密保存，另存 Hash 供查詢**

這一版我會視為目前的**薪資 Schema 基準版**。

---

## 2026/08/22 21:37:00

### 使用者

記錄起來

### 助理

已記錄，這一版作為目前**薪資模組的基準 Schema**。

之後如果我們繼續討論「勞健保、所得稅、加班費、請假扣薪、薪資計算公式」等，我會在這套結構上往下延伸，不會重新推翻目前已確認的設計。

