# 今日技術練習｜2026/11/06

**主題：ISTQB-CTFL × Decision Table Testing × Requirement Traceability**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜優惠券 API 的 8 個測試全數 PASS，為什麼仍發生不應有的折扣？

你負責驗證訂單結帳 API 的優惠券決策邏輯。所有數值、帳號與規則均為本題虛構 LAB 設定。優惠券 `SAVE100` 的折扣為新臺幣 100 元；本題不考慮其他優惠、稅額及運費。

### 一、已核准的業務規則

| ID | Requirement |
|---|---|
| REQ-01 | 只有當會員為 `ACTIVE`、購物車小計 `subtotal >= 1000`、優惠券 `valid=true`，且該會員**未使用過**此券（`used=false`）時，才能套用折扣 100 元。四個條件必須同時成立。 |
| REQ-02 | 不符合任一條件時，折扣必須為 0，且不得建立新的 Coupon Redemption Ledger 記錄。 |
| REQ-03 | 成功套用時，API 回傳 `discount=100`，且 Primary Redemption Ledger 必須恰好新增一筆與本次 `order_id` 關聯的已提交記錄。 |
| REQ-04 | Release Gate 必須以核准的規則及獨立的 Ledger 查詢作為驗證依據，不能只依 API 回應或既有自動化測試 PASS 判定。 |

### 二、既有測試

測試資料均為不同的會員與訂單，每筆開始前 Ledger 已清理至已知狀態。表中 `A` 表示會員 ACTIVE、`M` 表示小計達門檻、`V` 表示券有效、`U` 表示券已使用過。

| TC | A | M | V | U | API discount | Test Result |
|---|---|---|---|---|---:|---|
| 01 | T | T | T | F | 100 | PASS |
| 02 | F | T | T | F | 0 | PASS |
| 03 | T | F | T | F | 0 | PASS |
| 04 | T | T | F | F | 0 | PASS |
| 05 | T | T | T | T | 0 | PASS |
| 06 | F | F | T | F | 0 | PASS |
| 07 | F | T | F | T | 0 | PASS |
| 08 | T | F | F | T | 0 | PASS |

CI 報告：`8/8 PASS`。目前這 8 個案例都只比對 API `discount`，沒有查詢 Primary Redemption Ledger。

### 三、QA 的額外觀測

QA 使用一筆全新訂單 `ORD-909`，輸入如下：

```text
member_status = ACTIVE
subtotal      = 1200
coupon_valid  = false
coupon_used   = true
```

結果：

```text
POST /checkout -> HTTP 200
API discount = 100
API final_amount = 1100
Primary Redemption Ledger: 1 committed row for ORD-909
```

RD 認為：「既有 8 個測試都 PASS，表示主要條件已覆蓋。這個案例可能是測試資料污染，先不要判定功能有問題。」

## (1) Decision Table Analysis｜8/8 PASS 等於決策規則完整覆蓋嗎？

**A.** 依 REQ-01～REQ-03，建立一張**精簡的決策表（可合併等價規則並使用 `-` 表示 don't care）**，列出條件組合與預期的 `discount`、Ledger 新增筆數。請說明你合併規則的依據。

**B.** 既有 8 個測試對四個條件的 True／False 值都有出現，是否足以證明所有決策規則都已驗證？請具體指出現有測試的覆蓋證據與限制；不得只用「8/8 PASS」當作覆蓋率。

**C.** QA 額外觀測對 REQ-01～REQ-04 各提供什麼證據？請標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，並區分「規格符合性」與「資料污染是否為根因」。

## (2) Test Design｜只能新增兩個測試流程

你只能新增**兩個測試流程**；每個流程可包含多次 API 呼叫，但需保留可追溯的會員、訂單與 Ledger 證據。測試環境允許建立全新會員、優惠券及訂單，並可在呼叫前後查詢 Primary Ledger。

請設計兩個流程，優先驗證：

- **多個條件同時不成立**時，系統不應錯誤放行。
- **同一會員重複使用**優惠券時，API 與 Ledger 的一致性。
- 每個流程至少有一項不依賴 API `discount` 的獨立判準。

每個流程請填寫：

```text
Test Flow ID:
Preconditions / Data Isolation:
Input and Operation Sequence:
Expected API Results:
Independent Ledger Oracle:
PASS / FAIL Criteria:
Evidence to Preserve:
```

## (3) Release Gate｜測試全綠與實際違規如何取捨？

**A.** 目前選擇 `Go`、`Hold` 或 `No-Go`？請根據具體 Requirement 與證據說明，不得把「尚未證明唯一根因」當成「沒有規格違反」。

**B.** 假設 RD 修正後，既有 8 個案例仍全部 PASS，`ORD-909` 的同型輸入現在回傳 `discount=0`。你還需要哪**兩項最關鍵的獨立回歸證據**，才能重新評估 Release？請明確寫出來源與 PASS／FAIL 判準。

**C.** 若新測試能抓到舊版缺陷，但在新版 PASS，你會如何保存證據，證明它確實有能力偵測這次的錯誤，而不是僅在新版偶然通過？

---

**作答方式：** 請在本題對話分支依序回答 (1)～(3)。本次不提供答案或解析；作答後再逐項檢查決策表、測試覆蓋、證據判讀與 Release Decision。
