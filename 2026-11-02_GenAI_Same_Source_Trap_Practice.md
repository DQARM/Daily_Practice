# 今日技術練習｜2026/11/02

**主題：GenAI Testing × Same-Source Trap × Independent Test Oracle**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜AI 產生的 48 個測試全部 PASS，為什麼財務仍拒絕放行？

你是訂閱制平台的 QA，負責驗證新版帳單計算 API。RD 使用 GenAI 協助產生程式碼，QA 也使用 GenAI 產生自動化測試。測試在隔離 Staging 環境進行，付款服務使用模擬交易，不涉及真實金流。

### 一、已簽核的業務規格（BRD v4）

所有金額均為新臺幣整數元。`round_half_up(x)` 表示四捨五入至整數，遇到 `.5` 一律進位。

| ID | 驗收要求 |
|---|---|
| REQ-01 | 商品小計為 `P`；折扣 `D = min(round_half_up(P × 10%), 120)`；運費固定 `60`；營業稅 `T = round_half_up((P − D + 60) × 5%)`；應付總額 `A = P − D + 60 + T`。 |
| REQ-02 | API 回傳的 `tax`、`total`，以及已提交的 Invoice Ledger、模擬 Payment Capture 金額，都必須符合 REQ-01 且彼此一致。 |
| REQ-03 | Release Gate 必須包含由已簽核業務規格推導、且**不依賴產品實作邏輯**的預期結果；只通過由產品程式推導的測試，不足以宣告計算正確。 |

### 二、程式與 AI 測試產生流程

RD 提供下列實作邏輯摘要：

```python
D = min(round_half_up(P * 0.10), 120)
T = round_half_up((P + 60) * 0.05)
A = P - D + 60 + T
```

QA 將**這段程式碼及其輸出範例**交給 GenAI，要求它產生單元測試及 API 測試，但**沒有提供 BRD v4**。

CI 結果：

```text
AI-generated Unit Tests : 32/32 PASS
AI-generated API Tests  : 16/16 PASS
CI Status               : GREEN
```

抽查發現，部分 AI-generated Tests 直接使用以下公式計算 Expected Tax：

```python
expected_tax = round_half_up((P + 60) * 0.05)
```

### 三、額外驗證證據

財務部依 BRD v4 建立一份獨立試算表，並由另一位審核者核對公式。QA 取得同一批測試交易的觀測結果：

| 證據 | Case A | Case B |
|---|---:|---:|
| 商品小計 `P` | 1,000 | 1,800 |
| 財務試算表：應付總額 | 1,008 | 1,827 |
| API：`total` | 1,013 | 1,833 |
| API：`tax` | 53 | 93 |
| Primary Invoice Ledger：已提交總額 | 1,013 | 1,833 |
| 模擬 Payment Capture：金額 | 1,013 | 1,833 |

兩個 Case 都使用獨立 Invoice ID；所有量測均在交易提交後取得，未觀察到重試或重複扣款。**目前尚未取得 GenAI 產生測試時的完整 Prompt／輸入來源清單，也尚未執行修正後的 Regression Test。**

RD 認為：

> 48 個測試全部通過，而且 API、Ledger、Payment Capture 三者一致。財務試算表可能算錯，應該可以 Release。

---

## (1) Evidence Analysis｜48/48 PASS 究竟證明了什麼？

**A.** 請說明目前 48 個 AI-generated Tests 的 PASS 結果能支持哪些結論、不能支持哪些結論。請特別分析 **Same-Source Trap** 是否存在，以及你判斷的證據與限制。

**B.** 請依 BRD v4 **自行列出 Case A 與 Case B 的折扣 `D`、課稅基礎、營業稅 `T`、應付總額 `A`**，並與 API 觀測結果比較。請保留計算式。

**C.** 對 REQ-01、REQ-02、REQ-03，分別標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，並說明每項判定依據。請區分「不同系統數值一致」與「數值符合業務規格」。

## (2) Test Design｜只能新增兩個高鑑別度測試

假設 QA 只能新增**兩個自動化測試案例**，但每個案例都可以檢查 API、Primary Invoice Ledger 與模擬 Payment Capture。你可以建立新交易、清除 Staging 測試資料，並取得 BRD v4 與財務試算表；不能修改產品原始碼。

請設計兩個案例，讓它們能檢查**折扣與課稅順序**及**折扣上限**。每個案例請填寫：

```text
Test Case ID:
Test Objective:
Input / Preconditions:
Independent Oracle（來源及預期計算）:
Test Procedure:
Assertions / PASS-FAIL Criteria:
Evidence to Preserve:
```

另請回答：你如何避免新測試再次從產品程式或同一份 AI 產生的公式取得 Expected Result？

## (3) Release Gate｜修正後的 48/48 PASS 夠不夠？

**A.** 依目前證據，請選擇 `Go`、`Hold` 或 `No-Go`，並區分「是否符合驗收規格」與「是否已查明唯一根本原因」。

**B.** 假設 RD 修正程式後，原本 48 個 AI-generated Tests 仍全部 PASS，且兩個新案例的 API 回傳數值與財務試算表一致。你還會要求哪**兩項最關鍵的獨立驗證證據**，才能重新評估 Release？請寫出各自的來源、可追溯性及 PASS／FAIL 判準。

**C.** 你會如何證明新增的測試不是「剛好跟著修正後程式一起 PASS」，而是有能力偵測本次觀察到的計算錯誤？請提出一種可執行的驗證方式與判定標準。

---

**作答方式：** 請在本題的對話分支依序回答 (1)～(3)。本次不附答案或解析；收到作答後，再針對本題解析核心觀念、證據界線、常見誤區與排查策略。
