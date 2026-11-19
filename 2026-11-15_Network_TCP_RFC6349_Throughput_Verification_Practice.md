# 每日技術練習｜2026/11/15

**主題：Network / Performance × RFC 6349 × TCP Goodput × Independent Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜16 條 TCP 連線跑到 930 Mbps，單連線卻只有 610 Mbps

你是企業網路設備 QA，驗證一台 1 GbE WAN Router 的 TCP 傳輸能力。兩端 Linux 測試主機透過 DUT 連接，使用隔離 LAB；測試主機、DUT、測量儀器時鐘已校準。所有數值與門檻都是本題虛構驗收條件，不代表 RFC 6349 的通用合格標準。

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義 | 本題用途 |
|---|---|---|
| RFC 6349 | Framework for TCP Throughput Testing；TCP 吞吐量測試框架 | 用來組織 TCP 效能測試與判讀。 |
| DUT | Device Under Test；受測設備 | 本題的 WAN Router。 |
| Goodput | 有效資料吞吐量 | 單位時間內交付給應用程式的有效資料量，不包含重傳造成的重複資料。 |
| RTT | Round-Trip Time；往返時間 | 封包送出至收到對應確認的往返延遲。 |
| MSS | Maximum Segment Size；TCP 最大區段資料長度 | 單一 TCP segment 可承載的最大 TCP payload。 |
| RWND | Receive Window；接收視窗 | 接收端通告可接收資料量的流量控制機制。 |
| CWND | Congestion Window；壅塞視窗 | 發送端依壅塞控制維護的傳送上限。 |
| BDP | Bandwidth-Delay Product；頻寬延遲乘積 | 頻寬乘以 RTT，表示填滿路徑所需的在途資料量級。 |
| `iperf3 -P` | Parallel Streams；平行串流數 | 指定測試同時使用的 TCP 連線數。 |
| Retransmission | 重傳 | TCP 重新傳送未被確認的資料區段。 |
| Independent Oracle | 獨立測試判準 | 不只依賴 DUT Dashboard，而使用獨立端點與封包量測作驗證。 |

### 一、已核准驗收需求

| ID | Requirement |
|---|---|
| REQ-01 | 在 RTT 40 ± 2 ms、TCP MSS 1460 bytes、60 秒穩態量測下，**單一 TCP 連線**的應用層 Goodput 必須 ≥ 750 Mbps。 |
| REQ-02 | 相同條件下，16 條平行 TCP 連線的**合計** Goodput 必須 ≥ 900 Mbps。 |
| REQ-03 | 測試不得以調整 RTT、取消原有路徑限制或變更驗收門檻來宣告通過；須記錄 TCP window、重傳、主機資源與路徑封包證據。 |
| REQ-04 | Release Gate 須以可追溯的端點與獨立封包量測確認 REQ-01、REQ-02；不能僅以 DUT Dashboard 或 16 條連線結果代替單連線驗收。 |

### 二、測試拓樸與結果

```text
Linux Sender ── Tap A ── DUT (1 GbE WAN Router) ── Tap B ── Linux Receiver
     |                         |                           |
  iperf3 client          DUT Dashboard                iperf3 server
```

兩次測試均使用同一對主機、相同路徑與 TCP congestion control `CUBIC`。每次執行 `-t 65 -O 5`，統計最後 60 秒；`-P` 分別設為 1 與 16。接收端與封包擷取器確認沒有擷取溢位。

| 指標 | Test A：`-P 1` | Test B：`-P 16` |
|---|---:|---:|
| 端點應用層 Goodput | **610 Mbps** | **930 Mbps（合計）** |
| RTT 中位數 | 40.5 ms | 40.2 ms |
| 接收端通告 RWND（穩態） | 4 MiB | 每條連線 4 MiB |
| Sender TCP retransmissions | 1,240 | 310（合計） |
| Sender CPU 最高使用率 | 52% | 77% |
| Receiver CPU 最高使用率 | 48% | 74% |
| Tap A／Tap B 的 TCP 重傳觀測 | 均可見重傳 | 均可見重傳 |
| DUT Dashboard | `WAN utilization=67%` | `WAN utilization=97%` |
| iperf3 結束狀態 | Completed | Completed |

另外一份主機快照顯示 Test A 的 `cwnd` 在不同時間點介於 1.8～4.5 MiB；目前尚無逐 RTT 的 CWND／RTT／loss 對齊紀錄，也未證實重傳起因。RD 認為：「16 條已經達到 930 Mbps，表示 1 GbE Router 效能合格。單連線慢可能是 TCP 的正常現象，可以直接 Go。」

---

## (1) Evidence Analysis｜合計達標能代替單連線嗎？

**A.** 對 REQ-01～REQ-04 各標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，並說明哪些是已觀測事實、哪些只是推測。

**B.** 請計算本題 1 Gbps × 40 ms 的 BDP（以 bytes 或 MiB 表示），並比較 Test A 的 RWND 與 CWND 快照。這些數字能支持什麼判斷？又有哪些限制？

**C.** 610 Mbps、930 Mbps、重傳次數、CPU、DUT utilization 各能證明什麼、不能證明什麼？目前是否能唯一歸因於 DUT？

## (2) Verification Design｜最多兩個高鑑別度流程

設計**最多兩個**驗證流程，區分：H1＝DUT／路徑造成 loss 或 queueing；H2＝主機 TCP window、壅塞控制或 CPU 限制；H3＝量測／統計口徑差異。

每個流程請填寫：

```text
Test Flow ID:
Controlled Variables / Baseline:
Sender / Receiver / Tap Measurements:
Procedure:
Independent Oracle:
PASS / FAIL Criteria:
How Results Distinguish H1 / H2 / H3:
Evidence to Preserve:
```

不可修改驗收 RTT、MSS、Goodput 門檻，或僅以增加 `-P` 取代單連線測試。

## (3) Release Gate｜如何判斷與回歸？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`，說明符合性、未確認的根因與待補證據。

**B.** 假設 RD 修正後單連線 Goodput 達 790 Mbps、16 條達 940 Mbps，請提出兩項彼此獨立、可關聯同一測試版本與時間窗口的回歸證據及 PASS／FAIL 判準。

**C.** 設計一個隔離 LAB 中安全可還原的 **negative control（負向對照）**，驗證你的測試能攔下「16 條達標但單連線未達標」的版本；說明已知違規條件、預期偵測行為與復原方式。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪僅提供題目與必要名詞解釋，不附答案、提示性解答或解析；作答後再逐項批改證據判讀、驗證設計、常見誤區與排查思路。
