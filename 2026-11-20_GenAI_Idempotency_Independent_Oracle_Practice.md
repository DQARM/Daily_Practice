# 今日技術練習｜2026/11/20

**主題：GenAI Testing × Coverage Matrix × Independent Oracle × Same-Source Trap**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜AI 說 Test Case 100% 覆蓋，會員點數卻被扣了兩次

你負責驗證一個會員點數兌換 API。開發團隊使用生成式 AI 產生程式與 Test Case，另一個 AI 再審查測試案例。CI 顯示所有測試通過，但 QA 取得了額外的資料庫證據。

本題為**原創、合法隔離 LAB**；所有帳號、點數、識別碼與時間都是虛構資料，未連接真實金流或個資系統。每次請求均在前一筆資料庫交易提交後才送出，沒有並行請求。

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義 | 本題用途 |
|---|---|---|
| GenAI | Generative Artificial Intelligence；生成式人工智慧 | 用於產生程式、Test Case 與審查意見。 |
| API | Application Programming Interface；應用程式介面 | 本題透過 HTTP 接收點數兌換請求。 |
| Idempotency Key | 冪等識別鍵 | 由用戶端提供，用來讓系統辨認重複提交的業務請求。 |
| Side Effect | 副作用／狀態變更 | 例如扣除點數或新增資料庫紀錄。 |
| Ledger | 帳本／交易明細表 | 記錄點數變動事件；本題可由 QA 獨立讀取。 |
| Commit ID | 提交識別碼 | 用來識別資料庫中個別已完成的寫入交易。 |
| Test Oracle | 測試判準依據 | 用來判定實際結果是否符合核准需求的獨立依據。 |
| Coverage Matrix | 覆蓋矩陣 | 用來追蹤 Requirement、風險與 Test Case 的對應關係。 |
| Same-Source Trap | 同源驗證陷阱 | 產品、測試及審查可能共用同一份不完整假設或摘要。 |
| Mutation Test／Negative Control | 變異測試／負向對照 | 在隔離測試環境建立已知違規狀態，以檢查驗證流程能否發現。 |
| HTTP `201`／`409` | Created／Conflict；已建立／衝突 | 本 LAB 核准的兩種 API 回應狀態碼。 |
| Release Gate | 發布關卡 | 用驗證證據判斷產品版本是否符合發布條件。 |

### 一、獨立核准的需求

正式需求保存在 QA 唯讀的 `REQ-AUDIT-v2`，未提供給生成實作與測試案例的 AI。測試帳號為 `U-501`，初始餘額 **200 點**；單次兌換金額 30 點。冪等鍵以**同一帳號、同一鍵值、24 小時有效期內**判定。

| ID | Requirement |
|---|---|
| REQ-01 | 第一次使用新 Idempotency Key 且點數足夠時，回傳 HTTP `201`，扣除指定點數**一次**，並新增**一筆**對應 Ledger 扣點紀錄。 |
| REQ-02 | 在有效期內用**相同帳號、相同 Key、相同請求內容**重試，須回傳 HTTP `201` 與第一次相同的業務 `transaction_id`，但**不得再次扣點或新增 Ledger 紀錄**。 |
| REQ-03 | 有效期內使用相同帳號與 Key、但兌換金額不同，須回傳 HTTP `409`，且不得改變餘額或 Ledger。 |
| REQ-04 | Release Gate 必須具備獨立需求版本、API 原始回應、已提交的 DB 餘額／Ledger 前後狀態，以及相同測試執行的事件關聯；不能只依 AI 評分或 HTTP 狀態碼放行。 |

### 二、AI 產生的 Test Case 與 Coverage Matrix

RD 提供給程式產生 AI、測試產生 AI 與審查 AI 的，都是同一份 `DEV-SUMMARY-v3`。這份摘要只強調 API 回應規則，**沒有列出 Ledger 數量或重試不得再次扣點的持久化判準**。

| Test Case | 測試內容 | AI 實際比對欄位 | CI 結果 | AI 標記覆蓋 |
|---|---|---|---|---|
| TC-01 | 新 Key 第一次兌換 | HTTP `201`、`transaction_id` 存在 | PASS | REQ-01 |
| TC-02 | 相同 Key、相同金額再次提交 | HTTP `201`、`transaction_id` 與首次相同 | PASS | REQ-02 |
| TC-03 | 相同 Key、不同金額再次提交 | HTTP `409` | PASS | REQ-03 |

AI 測試報告寫著：`3/3 PASS; requirement mapping coverage = 100%`。第二個審查 AI 只讀取 `DEV-SUMMARY-v3`、上述 Test Case 與 CI 結果，也評為 `APPROVED`。目前沒有針對 REQ-04 的獨立檢核。

### 三、QA 的獨立觀測

以下三筆請求依序執行，均使用帳號 `U-501`，同一次 `run_id=QA-1120-01`；每筆 DB 快照都在該次請求**完整提交後**、下一筆請求發送前取得。

| 步驟 | HTTP 請求 | API 實際回應 | DB 餘額 | Ledger 扣點筆數 | 新增 Commit ID |
|---|---|---|---:|---:|---|
| 初始 | 尚未送出 | — | 200 | 0 | — |
| E1 | `key=K-42, amount=30` | `201`, `transaction_id=TX-901` | 170 | 1 | `C-101` |
| E2 | `key=K-42, amount=30` | `201`, `transaction_id=TX-901` | **140** | **2** | `C-102` |
| E3 | `key=K-42, amount=35` | `409` | 140 | 2 | 無 |

QA 的唯讀 DB Audit 另記錄：`C-101`、`C-102` 為兩筆不同的已提交扣點事件，均可關聯到 `run_id=QA-1120-01` 與 `U-501`。目前尚未取得完整程式呼叫堆疊，因此**不能單憑這些紀錄確定程式錯誤的唯一根因**。

RD 認為：「三個 Test Case 全部 PASS，審查 AI 也 APPROVED；E2 的 `transaction_id` 沒變，代表冪等成功。DB 可能只是測試環境的紀錄問題，可以先 Release。」

---

## (1) Evidence Analysis｜測試案例有對應需求，等於驗證了需求嗎？

**A.** 對 REQ-01～REQ-04 各標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，並引用最有鑑別度的觀測證據。請把「已證實的實際行為」、「尚未確定的根因」與「測試涵蓋不足」分開。

**B.** `3/3 PASS`、`requirement mapping coverage=100%`、審查 AI 的 `APPROVED`、相同 `transaction_id`，各自能證明什麼、不能證明什麼？本題的 Same-Source Trap 出現在什麼環節？

## (2) Verification Design｜只新增兩個測試流程

請設計**最多兩個**能重複執行的流程，驗證「同 Key 重試」與「同 Key 不同內容」的 API 與資料庫效果。你可以使用隔離帳號、DB Snapshot、API 原始紀錄、DB 唯讀查詢及獨立的 `REQ-AUDIT-v2`；不得改動需求、使用正式會員資料，或讓生成 Test Case 的 AI 自行決定所有 Expected Result。

每個流程請填寫：

```text
Test Flow ID:
Initial Balance / Ledger / Key State:
Request Sequence:
Expected HTTP and transaction_id:
Independent DB Oracle (balance, ledger, commit IDs):
Correlation and Isolation Controls:
PASS / FAIL Criteria:
```

另外，請說明你會如何更新 **Requirement × Test Case × Oracle** 覆蓋矩陣，避免只有「編號有連到 Test Case」就宣稱完整驗證。

## (3) Release Gate｜AI 審查通過是否足以放行？

**A.** 依目前證據選擇 `Go`、`Hold` 或 `No-Go`，簡述主要理由；不得將推測的根因寫成已確認的事實。

**B.** 假設 RD 修正後，AI 測試與審查依然全部 PASS。請提出**兩項不同來源、可對應同一個 `run_id` 的回歸證據**，各寫出獨立判準與 PASS／FAIL 條件。

**C.** 設計一個安全可復原的 **Negative Control／Mutation Test**，證明新增的驗證流程能攔下「HTTP 回應及 `transaction_id` 正確，但同一筆兌換仍重複扣點」的版本；寫出已知違規條件、預期測試反應及復原方式。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪只出題並提供必要名詞解釋，不附答案、提示性解答或解析；收到作答後，再逐項批改證據判讀、覆蓋矩陣、驗證設計與 Release Decision。
