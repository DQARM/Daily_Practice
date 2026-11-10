# 今日技術練習｜2026/11/09

**主題：Network / Performance × Queueing Latency × Independent Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜吞吐量達標、平均延遲正常，為何語音封包仍大量逾時？

你負責驗證隔離 LAB 中一台 1 GbE Router DUT 的 QoS。測試流量包含 UDP 語音與 TCP 大檔傳輸。以下皆為本題虛構數據與驗收門檻，不代表業界通用標準。

```text
Voice Generator ─┐
                 ├── DUT (1 GbE egress) ── Receiver / Hardware Timestamp Analyzer
TCP Generator ───┘
```

### 一、驗收需求

| ID | Requirement |
|---|---|
| REQ-01 | 在 TCP 背景流量持續存在時，語音 UDP 封包的 DUT **單向轉送延遲 p99 ≤ 10 ms**，以同一個 120 秒穩態窗口內的所有有效語音封包計算。 |
| REQ-02 | 同一窗口內，語音 UDP 丟包率 < 0.1%；分母為發送端實際送出的語音封包數，須排除測量系統自身遺失造成的假象。 |
| REQ-03 | TCP 應用層有效吞吐量 ≥ 700 Mbps，且語音 REQ-01、REQ-02 必須**同時**達標。 |
| REQ-04 | Release Gate 必須具備獨立於 DUT Dashboard 的延遲、封包數與時間同步證據。 |

語音產生器設定為每秒 1,000 個 UDP 封包；發送端每個封包都有遞增 sequence number。120 秒窗口的起訖由測試控制器標記，兩端硬體時間戳由共同時鐘參考同步。

### 二、現有觀測

| Evidence | Observation |
|---|---:|
| DUT Dashboard：Voice average forwarding latency | 2.1 ms |
| DUT Dashboard：Voice p99 latency | 7.8 ms |
| Receiver Analyzer：Voice p50 / p95 / p99 | 1.9 / 8.4 / **17.6 ms** |
| Sender：Voice packets transmitted | 120,000 |
| Receiver：Voice unique sequence numbers received | 119,820 |
| Receiver：Voice duplicate packets | 20（已從 unique count 排除） |
| TCP application throughput | 735 Mbps |
| DUT egress interface drops | 0 |
| DUT QoS queue peak occupancy | 92% |
| DUT Dashboard：QoS policy | `VOICE_HIGH` enabled |

**量測品質補充：** Analyzer 的硬體 timestamp 校正紀錄顯示發送端與接收端之間可能存在固定 **+3.0 ms** 的殘餘時間偏差（符號定義：接收端時間戳偏晚）。目前尚未確認該偏差是否已由 Analyzer 的延遲結果扣除；也尚未取得兩端時間同步狀態的逐秒紀錄。Receiver 的 sequence 統計已排除 duplicate，但尚未確認發送端與接收端統計是否嚴格對齊同一窗口，以及窗口尾端在途封包的處理方式。

RD 表示：「TCP 已有 735 Mbps，DUT 顯示 p99 只有 7.8 ms，而且介面 drops=0，QoS 應該 PASS。Analyzer 可能是時間不同步。」

## (1) Evidence Analysis｜漂亮的平均值與 p99

**A.** 對 REQ-01～REQ-04 分別標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，並說明每個判定依據及證據限制。不要把尚未校正的時間戳直接視為無爭議的真值。

**B.** 依現有 sender／receiver 計數計算**表面上的**語音丟包率（列出分子、分母與百分比）。它能否直接作為 REQ-02 的最終驗收結果？指出至少兩個需要釐清的計數／窗口問題。

**C.** `average=2.1 ms`、`DUT p99=7.8 ms`、`Analyzer p99=17.6 ms`、`egress drops=0`、`queue peak=92%` 分別能證明什麼、不能證明什麼？哪些資料可能互相矛盾，哪些其實不衝突？

## (2) Verification Design｜最多兩個驗證動作

請設計**最多兩個**資訊價值最高的驗證動作，區分以下假設：

| Hypothesis | Description |
|---|---|
| H1 | DUT 在 TCP 競爭下確實造成語音尾端延遲／丟包超標。 |
| H2 | Analyzer 時鐘偏差、校正或時間戳處理導致延遲被高估。 |
| H3 | Sender／Receiver 的統計窗口、在途封包或計數口徑造成丟包誤判。 |

每個動作請填寫：

```text
Verification 1 / 2:
Controlled Variables:
Measurement Points / Raw Fields:
Clock / Window Correlation:
Expected Evidence:
How Results Distinguish H1 / H2 / H3:
PASS / FAIL Criteria:
```

限制：不得降低 TCP 背景負載、改變語音封包速率或自行放寬門檻；不能只回答「抓封包」或「重新測試」。

## (3) Release Gate｜修正後數字變漂亮，如何避免誤放行？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`，依 REQ-01～REQ-04 說明。請區分「尚有量測爭議」與「已證實符合規格」。

**B.** 假設 RD 調整 QoS 後，DUT Dashboard 顯示 p99=5.5 ms、TCP=742 Mbps，但 Analyzer 仍未完成時鐘校正。你要求哪**兩項相互獨立、能追溯到同一 120 秒窗口**的證據，才會重新評估 Release？請寫出各自的來源與 PASS／FAIL 判準。

**C.** 假設完成校正後，Analyzer p99=9.8 ms，量測擴展不確定度為 ±0.6 ms，而驗收文件未規定 guard band。你會如何記錄結果、提出何種判定政策需求？不得把自行選定的政策當成已核准規格。

---

**作答方式：** 請在本題對話分支依序回答 (1)～(3)。本次不附答案、提示性解答或解析；作答後再逐項批改證據判讀、驗證設計與 Release Decision。
