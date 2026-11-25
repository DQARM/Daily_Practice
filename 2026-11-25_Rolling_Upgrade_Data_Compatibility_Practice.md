# 今日技術練習｜2026/11/25

**主題：軟體測試／驗證策略 × Rolling Upgrade × Data Compatibility × Release Gate**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜新舊版本都健康，為什麼同一筆設備設定會被讀成兩種結果？

你負責驗證企業網路設備的集中管理平台。平台由兩個服務節點共用一個資料庫，採滾動升級（Rolling Upgrade）：先將節點 B 升級至 v2，節點 A 暫時保留 v1。負載平衡器在升級期間仍會將請求分派至兩個節點。

v1 使用 `enabled` 布林欄位保存告警通知設定；v2 新增 `mode` 欄位，值為 `ON` 或 `OFF`。本次版本沒有新增其他通知模式。下列需求、資料、程式行為與紀錄都是原創虛構 LAB 資料，不涉及真實設備或客戶。

### 名詞與縮寫速查

| 名詞 | 英文全名／繁體中文意義 | 本題用途 |
|---|---|---|
| Rolling Upgrade | 滾動升級 | 逐一更新服務節點，使不同版本可能同時提供服務。 |
| Version Skew | 版本差異 | 描述同一系統中不同節點運行不同版本的狀態。 |
| Schema Migration | 資料結構遷移 | 調整資料庫欄位或資料格式，使應用程式可使用新的結構。 |
| Backward Compatibility | 向後相容性 | 描述新版本與既有資料或介面的相容能力。 |
| Rollback | 版本回退 | 將應用程式恢復至先前版本；不一定同時還原資料庫。 |
| Primary DB／Commit | 主要資料庫／交易提交 | 用於觀察已完成交易的持久化資料。 |
| HTTP 200 | OK；請求成功 | 本題 API 用來回報讀取或更新請求的成功狀態。 |
| `request_id`／`commit_id` | 請求識別碼／交易提交識別碼 | 用來關聯 API 操作與資料庫交易。 |
| Test Oracle | 測試判準依據 | 根據核准需求判定實際結果是否正確。 |
| Release Gate | 發布關卡 | 用來決定版本是否符合發布條件。 |

### 一、已核准需求

| ID | Requirement |
|---|---|
| REQ-01 | 更新通知設定成功並回傳 HTTP 200 後，下一筆讀取不論由 v1 或 v2 處理，均須反映最新已提交的業務設定。 |
| REQ-02 | 從 v1/v2 混合部署回退至全 v1 時，不得遺失已成功提交的設定；不得藉由還原舊資料庫快照來完成回退。 |
| REQ-03 | 滾動升級與回退期間，合法的設定讀寫請求均須成功；不得以停止服務或禁止舊節點寫入來滿足相容性需求。 |
| REQ-04 | Release Gate 須具備版本路由、獨立需求判準、API 原始回應、已提交的資料庫狀態及回退後的實際讀取證據。 |

所有讀寫均指同一筆設備設定 `device_id=LAB-27`。本次測試只有一個寫入者，請求依序執行；每次更新的交易提交後才發送下一筆請求。兩個節點讀取同一個 Primary DB。所有請求與交易均有完整識別碼，沒有其他背景寫入。

### 二、測試結果

升級前通知設定為 ON。遷移程序新增 `mode`，並將這筆既有設定轉換成 `mode=ON`。測試人員能指定請求由節點 A 或 B 處理。

| 步驟 | 節點／操作 | API 回應 | Primary DB 已提交狀態 |
|---|---|---|---|
| E0 | 混合部署完成 | A=v1、B=v2 | `enabled=true, mode=ON` |
| E1 | B/v2：更新為 OFF | 200，OFF | `enabled=true, mode=OFF`，C101 |
| E2 | B/v2：讀取 | 200，OFF | 同 C101 |
| E3 | A/v1：讀取 | 200，ON | 同 C101 |
| E4 | A/v1：更新為 OFF | 200，OFF | `enabled=false, mode=OFF`，C102 |
| E5 | A/v1：更新為 ON | 200，ON | `enabled=true, mode=OFF`，C103 |
| E6 | B/v2：讀取 | 200，OFF | 同 C103 |
| E7 | B/v2：更新為 OFF | 200，OFF | `enabled=true, mode=OFF`，C104 |
| E8 | 將 B 回退至 v1，不還原 DB | 兩節點健康 | 同 C104 |
| E9 | B/v1：讀取 | 200，ON | 同 C104 |

以下是同一個 `run_id=UP-1125-01` 的稽核紀錄節錄：

```text
09:00:01.100 request_id=R01 node=B version=v2 commit_id=C101 result=200
09:00:01.200 request_id=R02 node=B version=v2 read=OFF result=200
09:00:01.300 request_id=R03 node=A version=v1 read=ON  result=200
09:00:03.100 request_id=R07 node=B version=v2 commit_id=C104 result=200
09:00:08.000 node=B version=v1 health=READY
09:00:08.200 request_id=R09 node=B version=v1 read=ON result=200
```

CI 的 v1-only 與 v2-only 測試各自使用全新資料庫，只執行同版本讀寫，合計 48/48 PASS。健康檢查與本次所有 API 請求也均成功。尚未取得完整 SQL 執行紀錄、資料遷移程序與應用程式內部讀寫邏輯。

RD 表示：「兩個版本的單獨測試都通過，回退後節點也健康；混合版本只有短暫存在，不必阻擋發布。」

---

## 第 1 題｜證據判讀（30 分）

**A.** 對 REQ-01～REQ-04 各標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，並指出對應證據與結論適用範圍。

**B.** 分別寫出 E3、E6、E9 依需求應得到的業務設定，以及實際結果是否符合。HTTP 200、DB Commit 與健康檢查各自能證明什麼？目前是否足以確認唯一的程式根因？

## 第 2 題｜測試策略與版本覆蓋（45 分）

下一輪只有兩個測試流程的預算。你可還原隔離 LAB 的初始快照、指定請求節點、控制升級／回退時點、查詢唯讀 DB 與收集完整事件紀錄。不得在同一流程中藉由還原舊資料庫掩蓋已提交的設定變更。

**A.** 建立混合部署期間的「寫入版本 × 讀取版本」覆蓋矩陣，標記現有 E1～E7 已直接驗證的組合及其結果；只計算更新後、下一次寫入前執行的明確讀取，不把更新回應當成另一筆讀取。

**B.** 設計最多兩個可重複的流程，驗證所有版本讀寫組合、ON/OFF 雙向切換，以及保留已提交資料的版本回退。每個流程填寫：

```text
Test Flow ID:
Initial State / Version / DB State:
Upgrade / Write / Read / Rollback Sequence:
Independent Expected Results:
Evidence and Correlation Fields:
PASS / FAIL Criteria:
```

## 第 3 題｜發布關卡（25 分）

**A.** 根據目前證據選擇 `Go`、`Hold` 或 `No-Go`，並區分已觀測違規、未完成的驗證與仍待釐清的根因。

**B.** 假設 RD 修正後，兩個版本的 CI 仍全部 PASS。請提出兩組來源不同、可關聯同一測試執行的回歸證據，說明如何判定混合部署與回退均符合需求。再設計一個安全可還原的負向對照（Negative Control），確認測試能發現「所有 API 都回傳 200，但讀回的業務設定不一致」的情況。

---

**作答方式：** 請在本題對話分支回答第 1～3 題。首輪只出題，不附答案、提示性解答或解析；作答後再逐項批改、給分並說明核心觀念、常見誤區與排查思路。
