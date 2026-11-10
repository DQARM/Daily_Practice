# 今日技術練習｜2026/11/08

**主題：GenAI Testing × RAG × Evidence-grounded Answer Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜RAG 回答附了正確文件引用，卻給出錯誤的退款期限

你負責驗證企業客服 RAG 系統。系統會從版本化知識庫檢索政策片段，再產生答案與引用。以下皆為虛構 LAB 資料。測試時間固定為 **2026-11-08**，使用者所在市場為 **TW**，問題是：「我昨天收到商品，最晚幾天內可以申請一般退貨退款？」不涉及瑕疵品或法定例外。

### 一、已核准的驗收要求

| ID | Requirement |
|---|---|
| REQ-01 | 對 TW 市場、2026-11-01 起收到的商品，一般退貨申請期限為**收貨後 14 個曆日內**；不得引用已失效政策作為現行答案。 |
| REQ-02 | 回答必須引用足以支持期限的有效政策片段；引用文件存在、格式正確，不等於內容受到支持。 |
| REQ-03 | 若有效政策片段未成功進入模型上下文，系統不得自信給出期限；應明確表示無法確認並轉交人工。 |
| REQ-04 | Release Gate 必須包含獨立於生成答案的政策判準，以及可追溯的檢索、上下文、模型輸出與版本證據。 |

### 二、知識庫版本與執行紀錄

| Document | Market | Effective period | Content | Status |
|---|---|---|---|---|
| `POL-RET-v3` | TW | 2025-01-01～2026-10-31 | 一般退貨期限為收貨後 **7 個曆日** | 已失效，仍留在索引 |
| `POL-RET-v4` | TW | 2026-11-01 起 | 一般退貨期限為收貨後 **14 個曆日** | 已核准、生效 |
| `POL-RET-v4` | HK | 2026-11-01 起 | 一般退貨期限為收貨後 **10 個曆日** | 已核准、生效 |

本次 RAG Trace：

```text
request_id: RQ-1108-17
market: TW
received_at: 2026-11-07
retrieval_top_k: 3
retrieved:
  1. POL-RET-v3 / TW / score=0.94
  2. POL-RET-v4 / HK / score=0.91
  3. POL-RET-v4 / TW / score=0.88
context_sent_to_model:
  [POL-RET-v3 / TW] 7 calendar days
  [POL-RET-v4 / HK] 10 calendar days
  # POL-RET-v4 / TW was truncated by context budget
model_answer:
  "您可在收貨後 7 個曆日內申請一般退貨退款。"
citation:
  POL-RET-v3 / TW
```

CI 報告：`RAG evaluation: 96/100 PASS`。既有測試只檢查答案含有數字、引用的文件 ID 存在於索引，以及輸出 JSON schema 合法；未檢查政策生效日期、market 與引用內容是否支持答案。

RD 認為：「引用的確是 TW 文件，檢索也找到了 v4，96% 通過率足以 Release。」

## (1) Evidence Analysis｜找到、送進模型、引用、符合規格是四回事

**A.** 對 REQ-01～REQ-04 分別標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，逐項指出可支持判定的證據；區分已觀測的違規與尚未確認的根本原因。

**B.** 請分別解釋 `retrieved`、`context_sent_to_model`、`citation`、`96/100 PASS` 各能證明什麼、不能證明什麼。是否可以僅憑檢索結果中有 TW v4 就宣稱模型已獲得正確政策？

**C.** 若要確認此錯誤究竟主要來自檢索排序、上下文裁切、生成階段或驗證判準，現有 Trace 還有哪些限制？請說明哪些觀測已足以定位部分失效環節，哪些仍需實驗。

## (2) Test Design｜最多兩個高鑑別度驗證流程

你可在 LAB 固定模型版本與隨機種子、建立索引快照、控制檢索結果及上下文裁切，並取得經政策負責人核准的版本化政策表。不得修改產品程式碼。

設計**最多兩個測試流程**，分別涵蓋：

- **版本與市場衝突**：舊 TW 政策與新 HK 政策同時被檢索。
- **有效證據缺席**：TW v4 未進入模型上下文時的回應行為。

每個流程填寫：

```text
Test Flow ID:
Controlled Inputs / Policy Version:
Retrieval and Context Preconditions:
Procedure:
Independent Oracle and Its Provenance:
Assertions / PASS-FAIL Criteria:
Evidence to Preserve:
```

另請說明如何避免 Expected Answer 直接由同一個 RAG 系統、同一批模型輸出或其生成的測試案例推導，造成 **Same-Source Trap**。

## (3) Release Gate｜修正後答對一次，是否足以放行？

**A.** 依目前證據選擇 `Go`、`Hold` 或 `No-Go`，說明是否符合需求與是否找到唯一根因之間的區別。

**B.** 假設 RD 修正後，同一問題回答「14 個曆日」，引用 `POL-RET-v4 / TW`，CI 仍顯示 96/100 PASS。請提出**兩項相互獨立、可追溯至同一次 request 的回歸證據**，各自寫出來源及 PASS／FAIL 判準。

**C.** 請設計一個可執行的 **negative control 或 mutation test**，證明你的新驗證機制能抓出「答案看似合理、引用格式正確，但政策版本或市場錯誤」的問題。寫出預期的測試結果及失敗判準。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪不附答案、提示性解答或解析；作答後再逐項批改證據判讀、測試設計與 Release Decision。
