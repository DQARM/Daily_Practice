# 今日技術練習｜2026/10/31

**主題：ISTQB-CTFL × State Transition Testing × Test Oracle**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜一次性授權 Token 的測試覆蓋率已達 100%，為什麼還會有兩次成功回應？

你負責驗證一個在隔離 LAB 中運作的「一次性下載授權 Token」服務。每個 Token 只允許兌換一次，兌換成功會建立一筆下載權限（Grant）。本題的時間、API 與資料均為虛構的測試情境。

### 系統規格

Token 初始狀態為 `ISSUED`，建立時間為 `t0`，有效期限為 `t0 + 60.000 秒`。測試環境有可控制的單一時鐘；到期事件在時間 `now >= expires_at` 時生效，且在同一時間點的 API 請求之前處理。

| 目前狀態 | 事件／操作 | 規定行為 | 下一狀態 |
|---|---|---|---|
| `ISSUED` | `redeem`，尚未到期 | HTTP `200`，**新增並提交一筆 Grant** | `USED` |
| `ISSUED` | `revoke`，尚未到期 | HTTP `204`，不建立 Grant | `REVOKED` |
| `ISSUED` | `expire`（到期事件） | 不建立 Grant | `EXPIRED` |
| `USED` | `redeem` 或 `revoke` | HTTP `409`，不新增 Grant | `USED` |
| `REVOKED` | `redeem` 或 `revoke` | HTTP `409`，不新增 Grant | `REVOKED` |
| `EXPIRED` | `redeem` 或 `revoke` | HTTP `409`，不新增 Grant | `EXPIRED` |

**REQ-01：** 每個 Token 最多只能有一次成功的 `redeem`。即使請求同時到達，也最多只能回傳一次 HTTP `200`。

**REQ-02：** HTTP `200` 表示此次 `redeem` 成功建立並提交一筆新的 Grant；對已兌換 Token 的後續請求不能以 `200` 重複回覆。本題不允許將重試視為成功的冪等回覆。

**REQ-03：** `now >= expires_at` 時不得成功兌換。

**REQ-04：** 不論 API 回應如何，實際提交的 Grant 數量不得超過一筆。

### 現有測試

| TC | 測試步驟 | 觀測結果 |
|---|---|---|
| TC-01 | 新 Token，`t0+10s` 執行 `redeem` | `200`，狀態 `USED` |
| TC-02 | 新 Token，`t0+10s` 執行 `revoke` | `204`，狀態 `REVOKED` |
| TC-03 | 新 Token，時間推進至 `t0+60s`，觸發到期事件 | 狀態 `EXPIRED` |

三個測試全部 PASS。QA 報告寫著：`Valid State Transitions Coverage = 100%`。

### 額外觀測：並行兌換

另一個新 Token `TK-88`，在未到期時收到兩個幾乎同時發出的 `redeem` 請求。

```text
Token: TK-88
expires_at: 10:01:00.000

Request A  10:00:59.998  -> HTTP 200, grant_id=G-81
Request B  10:00:59.999  -> HTTP 200, grant_id=G-81

Primary Grant Ledger（查詢時）:
  committed grant rows for TK-88 = 1
  grant_id = G-81

Token final state = USED
```

RD 認為：「兩次都回同一個 `grant_id`，而且資料庫只有一筆 Grant，所以沒有問題；三條合法轉移也都測過了。」

---

## (1) Coverage Analysis｜100% 是哪一種 100%？

本題**明確採用以下計數口徑**：將三條 `ISSUED` 的合法狀態變更，以及三個終態各自對 `redeem`、`revoke` 的兩條拒絕規則，合計視為 **9 條 state/event rules**。不把未列出的事件納入分母。

**A.** 現有 TC-01～TC-03 的「合法狀態變更覆蓋率」是多少？若以全部 9 條 state/event rules 為分母，覆蓋率又是多少？請列出分子、分母。

**B.** 這兩種覆蓋率對產品品質的證明力有何差別？只觀察 HTTP Status 與最終 Token State，是否足以確認一次兌換規格？

## (2) Test Design｜最多三個測試流程

請設計**最多三個測試流程**（一個流程可包含多個 API 操作），盡可能同時覆蓋：

- `USED`、`REVOKED`、`EXPIRED` 三種終態的拒絕規則。
- 到期前最後一個有效測試時間點 `t0+59.999s`，以及到期邊界 `t0+60.000s`。
- 兩個同時兌換同一 Token 的競爭情境。

每個流程請交代：**初始狀態／時間控制、操作順序、預期 HTTP Status、預期 Token State、獨立的 Grant 驗證證據**。另請說明如何避免因測試時鐘不穩定而把邊界測試誤判為產品缺陷。

## (3) Evidence × Release Gate｜一筆 Grant、兩個 200

**A.** 額外觀測是否已足以證明 REQ-01～REQ-04 中有任何一項不符合？請逐項區分「已確認」「尚無足夠證據」，不要把 HTTP 回應與已提交 Grant 的數量混為一談。

**B.** 目前你會選擇哪一種 Release Decision？請以 Requirement 與證據說明理由。

- **Go**：只有一筆已提交 Grant，因此可放行。
- **Hold**：必須先證明資料庫產生兩筆 Grant，才能判定違規。
- **No-Go**：即使沒有兩筆 Grant，也可能已存在足以阻止放行的規格違反。

**C.** 如果 RD 修正後，這三個原有測試依然全部 PASS，你會要求補上哪一項**最有鑑別度的回歸驗證證據**，才能支持重新評估 Release？請提出明確的 PASS／FAIL 判準。

---

**作答方式：** 請在本題對話分支中回答 (1)～(3)。首輪不附答案或解析；收到作答後再針對該題解析核心原理、證據界線、常見誤區與排查策略。
