# 今日技術練習｜2026/11/07

**主題：軟體測試／驗證策略 × 非同步事件 × Exactly-once Business Effect**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜事件重送後，API 顯示成功，庫存卻被扣了兩次

你負責驗證一個訂單服務與庫存服務之間的非同步事件處理流程。以下數值、需求、日誌均為虛構的隔離 LAB 情境。訊息代理採 **at-least-once delivery**；同一個 `event_id` 可能被重送。每筆訂單只含一種 SKU，數量固定為 1。

```text
Client -> Order API -> Orders DB
                       |
                       v
                 Event Broker -> Inventory Consumer -> Inventory DB
                       ^                  |
                       +------ retry -----+
```

### 一、驗收要求

| ID | Requirement |
|---|---|
| REQ-01 | 每個已接受的 `OrderCreated`（以唯一 `order_id` 識別）對同一 SKU **最多造成一次**庫存扣減；重送不得再次扣減。 |
| REQ-02 | Consumer 僅在業務效果已持久提交後才可將事件標記為已完成；若處理失敗，允許重試，但不得遺失應有的扣減。 |
| REQ-03 | 對已接受且庫存充足的訂單，在沒有持續性故障時，庫存扣減須於 **30 秒內**完成。 |
| REQ-04 | Release Gate 必須包含獨立的資料庫前後狀態、事件關聯及故障注入證據；不得僅依 API `202`、Consumer `SUCCESS` 或 Broker ACK 判定。 |

### 二、測試步驟與觀測

測試開始前，SKU `SKU-7` 的 Inventory DB 可用庫存為 **50**，沒有其他流量。QA 發出一筆訂單 `ORD-71`，其事件識別碼為 `EVT-900`。

```text
10:00:00.000  POST /orders -> HTTP 202, order_id=ORD-71
10:00:00.120  Broker delivers EVT-900 (delivery #1)
10:00:00.180  Consumer log: UPDATE inventory SET available = available - 1
10:00:00.200  Fault injector: terminates consumer process before ACK
10:00:03.100  Broker redelivers EVT-900 (delivery #2)
10:00:03.180  Consumer log: processing SUCCESS
10:00:03.190  Broker ACK recorded
```

| Evidence | Observation |
|---|---|
| Inventory DB `available` before | 50 |
| Inventory DB `available` at 10:00:05 | 48 |
| Orders DB | `ORD-71` 存在，狀態 `ACCEPTED` |
| Broker | `EVT-900` delivery count = 2，最終 ACK |
| Consumer logs | 兩次 delivery 都有相同 `event_id`；第一次沒有 SUCCESS |
| Inventory audit table | 有兩筆 `SKU-7` 數量 -1 的異動，但未記錄 `order_id` 或 `event_id` |
| DB transaction commit / rollback trace | 尚未取得 |
| Deduplication record / unique constraint | 尚未確認 |

RD 認為：「Broker 重送是正常的 at-least-once 行為；第二次 SUCCESS 且 ACK，表示流程已恢復。第一次沒有 SUCCESS，不能算一次扣減，所以可以放行。」

## (1) Evidence Analysis｜SUCCESS、ACK 與業務效果

**A.** 對 REQ-01～REQ-04 各標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，並指出每項判定依據。請區分已觀測的業務結果、訊息傳遞狀態與尚未證明的因果關係。

**B.** `available: 50 -> 48`、兩筆 audit 異動、兩次 delivery、一次 ACK，分別能支持哪些結論？哪些結論還需要 transaction／event 關聯才能成立？

**C.** 你是否接受 RD 的放行理由？請分成「已確認事實」「合理假設」「尚不能下的結論」回答。

## (2) Test Design｜最多兩個故障注入流程

你可在 LAB 重建資料庫快照、控制 Broker 重送、在指定程式邊界終止 Consumer、查詢 Primary DB 與交易日誌；不能修改產品原始碼。

請設計**最多兩個**測試流程，驗證重送下的業務效果與失敗恢復。每個流程須填寫：

```text
Test Flow ID:
Initial DB / Broker State:
Fault Injection Point and Proof of Timing:
Event Identity and Correlation Fields:
Procedure:
Independent Oracle / Measurements:
PASS / FAIL Criteria:
```

另請說明：如何避免把「故障注入指令已送出」誤認為「Consumer 確實在指定交易階段被終止」？

## (3) Release Gate｜修正後 ACK 正常，還需要什麼？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`，說明是否符合規格與是否查明唯一根因之間的關係。

**B.** 假設 RD 修正後，同樣的兩次 delivery 最終只觀察到 `available: 50 -> 49`，且 Broker ACK 成功。請提出**兩項相互獨立、可追溯至 `ORD-71`／`EVT-900` 的回歸驗證證據**，各自列出 PASS／FAIL 判準。

**C.** 你會如何證明新增測試真的能偵測舊版的「重送後重複扣減」缺陷，而不是只在新版偶然 PASS？請提出可執行的方法與判定標準。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。本次不附答案或解析；作答後再逐項批改證據判讀、測試設計與 Release Decision。
