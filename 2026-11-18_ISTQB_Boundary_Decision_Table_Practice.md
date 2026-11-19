# 每日技術練習｜2026/11/18

**主題：ISTQB-CTFL × Decision Table Testing × Boundary Value Analysis**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜退款規則的測試全部 PASS，為什麼第 14 天的 VIP 訂單被錯誤拒絕？

你負責驗證隔離 LAB 中的訂單退款資格 API。系統依據申請日距收貨日的天數、商品是否未拆封，以及會員是否為 VIP，決定自動核准、人工審查或拒絕。所有資料均為虛構，且本題僅採用下列已核准業務規則。

### 名詞與縮寫速查

| 名詞 | 英文全名／中文說明 | 本題用途 |
|---|---|---|
| ISTQB-CTFL | Certified Tester Foundation Level；軟體測試基礎級認證 | 本題測試設計知識範圍。 |
| Decision Table | 決策表 | 將條件組合對應到預期業務動作。 |
| Rule Coverage | 規則覆蓋率 | 已測試決策規則佔全部已定義規則的比例。 |
| Don't Care（-） | 不影響決策的條件 | 用於合併決策表中結果相同的組合。 |
| BVA | Boundary Value Analysis；邊界值分析 | 選擇門檻附近的輸入資料。 |
| Test Oracle | 測試判準依據 | 用已核准需求建立預期結果。 |
| Mutation Testing | 變異測試 | 刻意修改判斷邏輯以檢查測試能否辨錯。 |
| API／HTTP 200 | Application Programming Interface／HTTP 成功回應 | API 介面及請求已處理的狀態，不直接定義業務決策。 |
| Release Gate | 發布關卡 | 決定版本是否符合驗收條件的檢查點。 |

### 一、已核准業務規則

API：

```http
POST /refunds/evaluate
Content-Type: application/json

{"days_since_delivery":14,"sealed":false,"vip":true}
```

回應格式：

```json
{"decision":"MANUAL_REVIEW"}
```

| ID | Requirement |
|---|---|
| REQ-01 | 收貨後第 0～14 天（含第 14 天）為期限內；第 15 天起為逾期。`days_since_delivery` 必須為非負整數。 |
| REQ-02 | 期限內：未拆封一律 `APPROVE`；已拆封且 VIP 為 `MANUAL_REVIEW`；已拆封且非 VIP 為 `REJECT`。 |
| REQ-03 | 逾期：未拆封且 VIP 為 `MANUAL_REVIEW`；其餘均 `REJECT`。 |
| REQ-04 | Release Gate 必須驗證各決策規則與 14／15 天邊界，預期結果來自核准需求，不得直接使用產品程式的輸出當 Oracle。 |

### 二、既有測試

每筆測試使用獨立訂單，API 回應均與測試腳本的 Expected Result 一致。

| TC | Days | Sealed | VIP | Expected | Result |
|---|---:|---|---|---|---|
| TC-01 | 7 | True | False | APPROVE | PASS |
| TC-02 | 7 | False | True | MANUAL_REVIEW | PASS |
| TC-03 | 7 | False | False | REJECT | PASS |
| TC-04 | 21 | True | True | MANUAL_REVIEW | PASS |
| TC-05 | 21 | True | False | REJECT | PASS |
| TC-06 | 21 | False | False | REJECT | PASS |

CI：`6/6 PASS`。尚未針對第 14、15 天設計專門測試，也未檢查測試 Expected Result 是否獨立於 RD 的程式碼。

### 三、QA 額外觀測

```text
Request ID: REF-1118-01
Input: days_since_delivery=14, sealed=false, vip=true
HTTP Status: 200
Response: {"decision":"REJECT"}
```

同一筆請求的輸入與回應已由外部 API Client 保存；測試資料沒有其他退款例外政策。RD 認為：「六個測試都 PASS，且 HTTP 200，這可能只是 VIP 資料判斷偶發問題，可以先 Release。」

## (1) Decision Table 與 Coverage

**A.** 根據 REQ-01～REQ-03，建立精簡 Decision Table。條件為 `within_14_days`、`sealed`、`vip`，動作為 `APPROVE`、`MANUAL_REVIEW`、`REJECT`。允許使用 Don't Care `-`，但規則必須互斥且完整。

**B.** 計算未合併前的 Boolean Condition Combinations 數量，以及合併後的決策規則數量。請指出 TC-01～TC-06 已覆蓋哪些規則、尚未覆蓋哪些規則；列出分子與分母。

**C.** HTTP 200 與 6/6 PASS 能證明什麼、不能證明什麼？對 REQ-01～REQ-04 分別標記 `PASS`、`FAIL` 或 `NOT VERIFIED` 並說明理由。

## (2) Boundary Value 與 Mutation Test

**A.** 只能新增**三個** `days_since_delivery` 測試值，你會選擇哪些值？請為每筆指定 `sealed`、`vip`、預期 `decision`，說明要檢驗的邊界。

**B.** 假設有人將 `days <= 14` 錯改成 `days < 14`。你新增的哪一筆測試最有能力偵測這項變異？請說明輸入、預期與觀察欄位，不需提供程式實作。

**C.** 若要防止 Same-Source Trap，應如何建立與版本化 Expected Result？請說明其來源與審核方式。

## (3) Release Gate

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`，並區分「已觀測到的業務規則違反」與「尚未查明的根因」。

**B.** 假設 RD 修正後，`REF-1118-01` 回傳 `MANUAL_REVIEW`，且原有 6 個測試仍全部 PASS。請提出**兩項最有鑑別度的回歸驗證證據**，寫出來源、關聯欄位及 PASS／FAIL 判準。

**C.** 如果測試預算只允許保留 8 個測試案例，你會如何選擇，兼顧 Decision Table Rule Coverage、14／15 天邊界與已知 Defect 的回歸偵測？請列出八筆以內測試輸入與預期結果。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。本次只提供題目與名詞解釋，不附答案、提示性解答或解析。
