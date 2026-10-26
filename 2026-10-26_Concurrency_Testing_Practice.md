# 今日技術練習｜2026/10/26

**主題：Software Testing × Concurrency Testing × Independent Verification**  
**難度：★★★★☆｜實務情境題｜預估 15–20 分鐘**

## Q1｜API 測試全部 PASS，庫存也沒有變成負數，為什麼仍可能發生超賣？

你負責驗證一套電子商務系統的訂單 API。

RD 宣稱已經實作庫存保護機制，所有 Unit Tests 與 Integration Tests 都通過。

但在 Production-like Environment 執行 Concurrent Request Test 時，你發現異常。

---

## 一、System Requirements

### REQ-01｜Inventory Consistency

商品庫存不得小於 0。

當商品剩餘庫存為 1 時，最多只能有一筆訂單成功保留該商品。

### REQ-02｜Order Confirmation

API：

```http
POST /api/orders
Content-Type: application/json

{
  "productId": 1001,
  "quantity": 1
}
```

Response：成功保留庫存：

```http
HTTP/1.1 201 Created
```

庫存不足：

```http
HTTP/1.1 409 Conflict
```

**本題規格明確定義：HTTP 201 代表庫存保留交易已成功提交，不是單純接受請求。**

### REQ-03｜Concurrent Requests

當多個使用者同時購買同一商品時，系統仍必須符合 REQ-01 與 REQ-02。

---

## 二、QA 測試結果

### Test A｜Sequential Requests

初始庫存：

```text
Product ID : 1001
Stock      : 1
```

依序執行兩次購買：

| Request | HTTP Status | Stock After Request |
|---|---|---:|
| User A | 201 Created | 0 |
| User B | 409 Conflict | 0 |

結果：

```text
Unit Tests        : 180/180 PASS
Integration Tests : 65/65 PASS
Sequential Test   : PASS
CI Status         : GREEN
```

RD 認為庫存保護機制正常。

### Test B｜Concurrent Requests

QA 將庫存重設為 1。

接著讓兩個不同使用者幾乎同時發送購買請求。

```text
                Initial Stock = 1
                       │
              ┌────────┴────────┐
              │                 │
           User A             User B
              │                 │
         POST /orders      POST /orders
              │                 │
              └────────┬────────┘
                       │
                   Order API
                       │
                    Database
```

測試結果：

| Measurement | User A | User B |
|---|---|---|
| Request Start | 10:00:00.100 | 10:00:00.101 |
| HTTP Status | 201 Created | 201 Created |
| Order ID | 5001 | 5002 |
| Response Time | 85 ms | 92 ms |

測試結束後，QA 查詢 Reporting Database：

```text
Product ID : 1001
Stock      : 0

Order 5001 : CONFIRMED
Order 5002 : CONFIRMED
```

但目前尚未確認 Reporting Database 是否存在 Replication Lag，也沒有取得 Primary Database 的 Transaction Log。

RD 回應：

> 庫存沒有變成負數，代表庫存保護機制正常。兩筆訂單都成功，可能只是 Reporting Database 的資料同步問題。

---

## (1) Evidence Analysis｜哪些結論已經成立？

請回答：

**A.** Sequential Test PASS，是否足以證明 Concurrent Requests 的行為正確？

**B.** Database Stock = 0，是否足以證明系統沒有發生超賣？

**C.** 兩個 HTTP 201 Response，是否已經構成違反 REQ-01／REQ-02 的證據？

請特別區分：

- API 對外宣告的結果。
- Database 的實際交易狀態。
- Reporting Database 可能造成的觀測偏差。

**D.** 目前能否直接判定根本原因是 Race Condition？

請至少提出兩個合理假設，並指出還缺少哪些證據。

---

## (2) Test Design｜如何設計具有鑑別度的並行測試？

目前 QA 的 Concurrent Test 只執行了一次。

RD 認為：

> 兩個 Request 只差 1 ms，不能證明真的同時進入庫存判斷邏輯。這次可能只是偶發事件。

假設你可以使用：

```text
pytest
Python threading / multiprocessing
HTTP Client
Database Read-only Access
Application Logs
```

但不能修改產品程式碼。

請設計一個可重複執行的 Concurrent Test。

填寫：

```text
Test Objective:

Initial State:

Test Procedure:

Concurrency Control Method:

Expected Results:

Independent Verification Evidence:
```

**額外要求：**

1. 必須避免前一次測試留下的訂單影響下一次結果。
2. 不得只依賴 HTTP Status 判定 PASS。
3. 必須能夠區分「兩個 Request 同時發出」與「兩個 Request 的關鍵交易區間實際重疊」。
4. 如果測試執行 100 次都 PASS，請說明這項結果能支持什麼結論，以及仍然無法證明什麼。

---

## (3) Troubleshooting × Release Gate

目前有三個假設：

| Hypothesis | Description |
|---|---|
| H1 | 庫存檢查與扣減不是 Atomic Operation |
| H2 | API 在 Database Transaction Commit 前就回傳 201 |
| H3 | Primary Database 正常，但 Reporting Database 的資料不一致 |

假設你只能再執行**兩個驗證動作**。

請選擇最有資訊價值的兩個動作，並回答：

```text
Verification 1

驗證位置：

要取得的證據：

不同觀測結果如何區分 H1、H2、H3：
```

```text
Verification 2

驗證位置：

要取得的證據：

不同觀測結果如何區分 H1、H2、H3：
```

### Release Decision

目前：

```text
Sequential Tests : PASS
Concurrent Test : 2 × HTTP 201
Reporting DB    : 2 CONFIRMED Orders
Final Stock     : 0
```

你會選擇：

**A. Go** — 庫存沒有變成負數，正式測試也全部通過。

**B. Hold** — 已觀察到異常，但必須先確認 Database Transaction 與 API Response 的關係，才能決定是否放行。

**C. No-Go** — 現有證據已足以證明對外行為違反明確的並行訂單規格，不應直接放行。

請選擇一個，並說明你的判斷依據。

---

## 本題考察重點

| 領域 | 核心能力 |
|---|---|
| Software Testing | Concurrency Testing |
| Database | Transaction、Atomicity、Isolation |
| Test Design | Race Condition Reproduction |
| Verification Strategy | Independent Verification Evidence |
| Evidence Analysis | API Response vs Database State |
| Release Gate | Go / Hold / No-Go |

### 本題核心挑戰

這次特別考驗三個能力：

**第一，** 如何區分「測試沒有發現錯誤」與「系統不存在並行問題」。

**第二，** 如何設計真正能提高並行衝突發生機率、而且能重複驗證的 Test Case。

**第三，** 如何根據不同來源的證據做出 Release Decision，而不是等 RD 找到 Root Cause 才決定產品是否符合需求。

**本次先不提供答案。** 請在這一題的對話分支中作答，再進行正解、核心觀念、常見誤區與排查思路解析。
