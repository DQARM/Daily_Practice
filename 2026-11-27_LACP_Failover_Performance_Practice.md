# 今日技術練習｜2026/11/27

**主題：Network / Performance × LACP × Link Aggregation × Failover**  
**難度：★★★★☆｜預估 10–15 分鐘｜滿分 100 分｜首輪只出題**

## 情境｜兩條 1 Gbps 網路線，拔掉一條後總吞吐量正常，為什麼部分連線仍然中斷？

你負責驗證企業交換器的鏈路聚合功能。兩台交換器以兩條 1 Gbps 全雙工實體鏈路組成 LAG，使用 LACP 協商。測試端持續發送四條 UDP 流量，每條使用不同的來源連接埠。RD 認為聚合介面始終顯示 UP，故障後的總吞吐量也符合要求，因此建議發布。

本題為原創隔離 LAB，設備、數據與驗收門檻均為虛構，不代表 IEEE 或 RFC 的通用合格標準。

```text
Sender ── Switch A (DUT) ══ Link 1 (1 Gbps) ══ Switch B ── Receiver
                     ╚════ Link 2 (1 Gbps) ════╝
                        LACP / LAG 10
```

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義 | 本題用途 |
|---|---|---|
| LAG | Link Aggregation Group；鏈路聚合群組 | 將多條實體鏈路組成一個邏輯介面。 |
| LACP | Link Aggregation Control Protocol；鏈路聚合控制協定 | 交換鏈路聚合成員的協商與狀態資訊。 |
| DUT | Device Under Test；受測設備 | 本題的 Switch A。 |
| Flow Hash | 流量雜湊 | 根據封包欄位選擇聚合群組中的傳輸成員。 |
| Failover | 故障切換 | 成員鏈路失效後，服務轉移至仍可用路徑的過程。 |
| UDP／Goodput | User Datagram Protocol；使用者資料包協定／有效吞吐量 | 本題使用 UDP 傳輸，以接收端有效應用資料量評估效能。 |
| `flow_id`／`seq` | 流量識別碼／封包序號 | 用來逐流量關聯傳送與接收事件。 |
| Tap／PCAP | Test Access Point／Packet Capture；線路分接點／封包擷取檔 | 用於保存實體鏈路上的封包證據。 |
| `oper_state`／`active_members` | 運作狀態／有效成員數 | 本題 DUT 儀表板提供的邏輯介面與成員狀態欄位。 |
| Release Gate | 發布關卡 | 依核准需求及驗證證據評估能否發布。 |

## 一、已核准需求

四條流量 F1～F4 各以固定 180 Mbps 應用資料速率持續傳送，每 1 ms 發送一批資料，每個封包均含唯一序號。已確認含協定開銷的總線路負載低於單一成員容量；端點也已通過獨立的容量基準測試。單一成員故障不應導致持續性過載。

| ID | Requirement |
|---|---|
| REQ-01 | 任一成員實體鏈路中斷時，LAG 邏輯介面須維持 UP，且每條既有流量在接收端的最大相鄰資料到達間隔不得超過 100 ms。 |
| REQ-02 | 故障後第 2～12 秒的穩態窗口內，每條流量的接收端 Goodput 均須達 175 Mbps，四條合計須達 700 Mbps。 |
| REQ-03 | 故障成員恢復後 5 秒內須重新加入可轉送成員集合；恢復期間每條既有流量的最大相鄰資料到達間隔仍不得超過 100 ms。 |
| REQ-04 | Release Gate 須保留兩條成員各自的故障及恢復測試、逐流量接收紀錄、兩端成員狀態與線路封包證據，並關聯至同一測試版本。 |

## 二、測試觀測

本次 `run_id=LAG-1127-A` 僅中斷 Link 1。外部測試儀記錄實際斷線時刻為 t0，於 t0+20 秒恢復鏈路。所有時間已對齊，間隔量測的擴展不確定度為 ±2 ms。接收端原始序號及時間戳已保存。

| Flow | 故障前成員（Tap 觀測） | 故障期間最大接收間隔 | 故障後穩態 Goodput | 恢復期間最大接收間隔 |
|---|---|---:|---:|---:|
| F1 | Link 1 | 420 ms | 179 Mbps | 55 ms |
| F2 | Link 1 | 380 ms | 178 Mbps | 60 ms |
| F3 | Link 2 | 4 ms | 180 Mbps | 4 ms |
| F4 | Link 2 | 5 ms | 180 Mbps | 5 ms |

故障期間最大間隔統計窗口為 t0 前 1 秒至 t0 後 12 秒；恢復期間的窗口為恢復前 1 秒至恢復後 6 秒。每條流量均持續發送，沒有重新建立流量或改變速率。

```text
DUT dashboard:
 t0-1.000s   oper_state=UP  active_members=2
 t0+0.018s   oper_state=UP  active_members=1
 t0+1.000s   oper_state=UP  active_members=1
 t0+20.000s  physical Link 1 restored (external event)
 t0+23.200s  oper_state=UP  active_members=2

CI summary:
 LAG_UP_AT_SAMPLES = PASS
 AGGREGATE_GOODPUT = 717 Mbps, PASS
 LINK_REJOIN = PASS
```

補充：故障後的穩態窗口內，Link 2 的 Tap 已觀察到四條流量。兩條實體鏈路在切換期間的完整 PCAP、Switch B 的成員狀態及恢復後的實際轉送證據尚未取得；DUT 的介面狀態為離散取樣，並非連續狀態事件。Link 2 單獨故障的情境尚未測試。

RD 表示：「LAG 沒有顯示 DOWN，717 Mbps 也超過 700 Mbps，Link 1 恢復後成員數又回到 2，Failover 已經 PASS。」

---

## 第 1 題｜效能與證據判讀（35 分）

**A.** 對 REQ-01～REQ-04 分別標記 PASS、FAIL 或 NOT VERIFIED，說明證據及結論適用範圍。對最大接收間隔的 ±2 ms 不確定度應如何處理？

**B.** 717 Mbps、`oper_state=UP`、`active_members=2` 與逐流量接收間隔各能證明什麼、不能證明什麼？單一流量使用兩成員的 LAG，是否必然能達到 2 Gbps？請說明理由。

**C.** 目前能否確定 420 ms 間隔的唯一根因是 LACP 協商太慢？還需要哪些時間點與封包證據，才能區分成員狀態轉換、實際流量轉送及端點接收行為？

## 第 2 題｜故障切換驗證設計（40 分）

下一輪只有兩個測試流程的預算。可控制任一實體鏈路的中斷與恢復、選擇四條流量的連接埠、取得兩端狀態事件及外部 Tap 紀錄。不得降低流量速率、重啟流量來掩蓋中斷，或放寬驗收門檻。

請設計最多兩個可重複的測試流程，涵蓋兩條成員各自的故障及恢復，並確保故障前的流量分布確實受到驗證。每個流程填寫：

```text
Test Flow ID / Initial State:
Traffic and Member Distribution:
Failure / Recovery Trigger:
Measurement Points / Time and Sequence Correlation:
Independent Expected Results:
PASS / FAIL Criteria:
Evidence to Preserve:
```

## 第 3 題｜發布關卡與負向對照（25 分）

**A.** 根據目前證據選擇 Go、Hold 或 No-Go，區分需求符合性與尚未確認的故障根因。

**B.** 若修正版本的總吞吐量仍為 717 Mbps，請提出兩組來源不同、可關聯同一 `run_id` 的回歸證據與判準。再設計一個安全可復原的負向對照（Negative Control），確認測試能發現「總吞吐量合格，但其中一條既有流量中斷超過門檻」的情況，寫出受控條件、預期測試反應及復原方式。

---

**作答方式：** 請在本題對話分支回答第 1～3 題。首輪只出題，不附答案、提示性解答或解析；作答後再逐項批改、給分，並說明核心觀念、常見誤區與排查思路。
