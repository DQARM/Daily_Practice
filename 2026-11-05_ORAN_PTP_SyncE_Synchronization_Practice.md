# 今日技術練習｜2026/11/05

**主題：5G／O-RAN × PTP／SyncE × Synchronization Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜PTP 顯示 LOCKED，為什麼 TDD Cell 仍然無法通過同步驗收？

你是 O-RAN 系統 QA，驗證隔離 LAB 中的 O-DU 與兩台 O-RU。兩台 O-RU 服務相鄰的 TDD Cell，共用一台 PTP Grandmaster（GM），透過支援 SyncE 的交換器連接。所有數據與門檻都是本題虛構的 LAB 規格，非通用 3GPP／O-RAN 標準限值。

```text
                     PTP GM（GNSS disciplined）
                              │
                      Switch（PTP + SyncE）
                         /             \
                      O-RU A          O-RU B
                         \             /
                               O-DU
                  （O-DU 與 O-RU 另有 C/U-Plane 連線）
```

### 一、驗收需求

| ID | Requirement |
|---|---|
| REQ-01 | 在穩態時，兩台 O-RU 相對 GM 的絕對時間偏差各不得超過 **±100 ns**；且 A、B 間的時間差絕對值不得超過 **150 ns**。 |
| REQ-02 | 兩台 O-RU 的頻率偏差絕對值各不得超過 **50 ppb**（相對於核准的頻率參考）。 |
| REQ-03 | 當 GM 失去 GNSS 參考時，系統須在 **10 秒內**產生可稽核的同步告警；不得僅因 PTP Port State 為 SLAVE 或 LOCKED 就宣告同步符合要求。 |
| REQ-04 | 驗收需包含獨立量測、量測不確定度及事件時間關聯，不能只依設備 Dashboard 的狀態。 |

### 二、測試數據

在 GNSS 正常的穩態 60 秒窗口，QA 取得以下結果。PTP Dashboard 的 offset 是各 O-RU 內部 servo 報告值；外部 Time Interval Analyzer（TIA）則透過兩台 O-RU 的 1PPS 測試輸出對 GM 參考 1PPS 量測，已校正線纜延遲。

| Measurement | O-RU A | O-RU B |
|---|---:|---:|
| PTP Port State | SLAVE | SLAVE |
| Dashboard Servo State | LOCKED | LOCKED |
| Dashboard `offsetFromMaster`（60s 平均） | +18 ns | +24 ns |
| 外部 TIA：對 GM 的時間偏差（60s 最大絕對值所對應的帶符號讀數） | +72 ns | +184 ns |
| 外部頻率計：相對核准參考的頻率偏差 | +12 ppb | +68 ppb |
| SyncE ESMC／SSM 狀態 | QL-PRC | QL-PRC |

**量測限制：** 外部 TIA 的每筆時間偏差讀數，其擴展量測不確定度為 **±15 ns**；外部頻率計的每筆頻率偏差讀數，其擴展量測不確定度為 **±5 ppb**。本題不預設任何特定 guard band 驗收規則。A、B 表中各自的最大值可能發生在不同時刻；尚未提供兩台 O-RU 同步取樣的逐筆時間序列。

### 三、GNSS 失鎖注入

QA 在 `14:00:00.000` 斷開 GM 的 GNSS 參考，並以獨立事件記錄器記錄斷開動作。

```text
14:00:00.000  External logger: GNSS reference disconnected
14:00:02.000  GM log: entered holdover
14:00:12.500  NMS: synchronization warning received
14:00:15.000  O-RU A dashboard: PTP LOCKED
14:00:15.000  O-RU B dashboard: PTP LOCKED
```

NMS 記錄的是**收到告警的時間**，目前尚未取得 GM 告警實際產生的 timestamp，也尚未核對 GM／NMS 的時鐘偏差及告警傳輸延遲。

RD 認為：「兩台 O-RU 都是 PTP LOCKED，SyncE 也顯示 QL-PRC；TIA 與 Dashboard 有差異可能是儀器誤差。NMS 在 12.5 秒才收到告警，也不一定代表設備晚告警，所以全部應該 PASS。」

## (1) Evidence Analysis｜LOCKED、時間與頻率是同一件事嗎？

**A.** 逐項說明 `PTP SLAVE`、`Servo LOCKED`、`offsetFromMaster`、`SyncE QL-PRC` 各能支持哪些判斷、不能直接證明哪些驗收要求。

**B.** 依目前數據，對 REQ-01 的「每台對 GM 偏差」與「A/B 相對偏差」，以及 REQ-02，分別給出 `PASS`、`FAIL` 或 `NOT VERIFIED`，說明量測不確定度與非同步最大值如何影響判定。不得直接用 +72 ns 與 +184 ns 相減當成已量測的最大 A/B 時間差。

**C.** 依現有事件紀錄，REQ-03 是否已被證明 PASS 或 FAIL？請分開說明「已觀測事實」與「缺少的時間關聯證據」。

## (2) Verification Design｜只能增加兩個驗證動作

請在下列假設之間設計**兩個資訊價值最高**的驗證動作：

| Hypothesis | Description |
|---|---|
| H1 | O-RU B 的實際時間／頻率同步品質不符合規格。 |
| H2 | Dashboard、外部儀器、測試輸出或量測參考的口徑／校正造成差異。 |
| H3 | GNSS 失鎖告警的產生、傳遞或時間戳關聯存在問題。 |

每個動作請包含：

```text
Verification 1 / 2:
Controlled Variables:
Measurement Points / Instruments / Logs:
Timestamp or Reference Correlation:
Expected Observations:
How Results Distinguish H1 / H2 / H3:
```

限制：不得更改驗收門檻；不能只回答「重新量測」或「檢查 Log」；必須指出要比對的資料欄位、時間基準與 PASS／FAIL 證據。

## (3) Release Gate｜不確定度與未完成證據如何處理？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`，逐項列出 REQ-01～REQ-04 的證據狀態。請區分「已觀測到的規格問題」「尚未驗證」及「尚未查明的根因」。

**B.** 假設 RD 修正後，O-RU B 外部量測的最大時間偏差為 **+94 ns（±15 ns）**，頻率偏差為 **+43 ppb（±5 ppb）**。在沒有預先定義 guard band 規則的情況下，你會如何記錄這兩項量測與驗收結論？請提出你會要求事先明確化的判定政策，但不要自行把政策當成既定規格。

**C.** 請提出一項能獨立驗證 **A/B 同步相對時間差** 的證據，以及一項能獨立驗證 **10 秒告警要求** 的證據；各自寫出可追溯性與 PASS／FAIL 判準。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪不提供答案或解析；作答後再逐項批改證據判讀、測試設計及 Release Decision。
