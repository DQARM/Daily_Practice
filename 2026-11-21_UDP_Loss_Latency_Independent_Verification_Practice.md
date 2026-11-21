# 今日技術練習｜2026/11/21

**主題：Network / Performance × UDP Loss × One-way Latency × Independent Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜交換器顯示丟包 0.06%，為什麼應用程式實際漏收 0.20%？

你負責驗證一條企業設備的 UDP 資料傳輸路徑。測試流量由 Sender 經 DUT（受測網路設備）送至 Receiver。RD 認為：只要交換器出口丟包率低於 0.1%，就可以宣告效能合格；QA 則取得了接收端應用程式的其他觀測。

本題所有拓撲、位址、封包、計數器與門檻均為**原創隔離 LAB 的虛構資料**，不是通用 RFC 效能標準。測試使用固定流量設定，沒有其他測試流量；所有量測紀錄均附同一個 `run_id=NET-1121-01`。

```text
Sender ── [Tap-A] ── DUT ── [Tap-B] ── Receiver NIC ── UDP Socket ── Application
  │                    │                      │                       │
  └── 發送紀錄          └── DUT Counters       └── Host Counters        └── 接收紀錄

Tap-A / Tap-B 使用硬體時間戳；兩端與 Receiver 的時間已校正。
```

### 名詞與縮寫速查

| 名詞 | 英文全名／繁體中文意義 | 本題中的用途 |
|---|---|---|
| UDP | User Datagram Protocol；使用者資料包協定 | 本題傳送具序號的資料封包，不由 UDP 自動保證交付或順序。 |
| DUT | Device Under Test；受測設備 | 本題在 Sender 與 Receiver 之間轉送 UDP 封包。 |
| NIC | Network Interface Controller；網路介面控制器 | Receiver 的網路介面，位於線路和作業系統之間。 |
| Tap／PCAP | Test Access Point／Packet Capture；線路分接點／封包擷取檔 | 在不依賴 DUT 自報數值的條件下保存線上封包。 |
| Sequence Number | 序號 | 每個原始 UDP 封包都有唯一遞增序號，重複副本使用同一序號。 |
| Unique Packet／Duplicate | 不重複封包／重複封包 | 用來區分收到幾個不同序號，及同一序號是否被收到多次。 |
| Socket Receive Buffer | Socket 接收緩衝區 | 作業系統供接收程式暫存 UDP 資料的佇列空間。 |
| `RcvbufErrors` | Receive Buffer Errors；接收緩衝區錯誤計數 | Linux UDP 統計欄位，表示接收緩衝區相關的封包丟棄事件。 |
| One-way Latency／p99 | 單向延遲／第 99 百分位數 | 用來評估指定測試窗口內大部分封包的到達時間分布。 |
| Capture Drop | 擷取遺失 | 量測設備未成功記錄經過的封包，不等於已證明網路丟包。 |
| Independent Oracle／Release Gate | 獨立測試判準／發布關卡 | 分別用於判定需求符合性，以及根據證據決定能否放行。 |

### 一、已核准驗收需求

本次測試在 **60 秒穩態窗口**內由 Sender 依序發出 **50,000 個不同序號**的 UDP 封包。每筆封包包含 `run_id`、`seq` 與可關聯的發送時間。除非特別註明，所有封包統計均限定這 50,000 個原始序號與同一窗口。

| ID | Requirement |
|---|---|
| REQ-01 | Receiver **應用程式**在窗口結束及核准的 1 秒收尾期後，至少須收到 **99.90% 不同序號**（端到端遺失率 ≤ 0.10%）；重複副本不可計入有效交付數。 |
| REQ-02 | 以 Sender 發送時間到 Receiver **應用程式收取時間**計算的單向延遲，`p99` 必須 **≤ 12.0 ms**；只對實際收到的原始序號計算，每個序號以首次到達時間計。 |
| REQ-03 | Release Gate 必須保留可按 `run_id` 與 `seq` 關聯的 Sender、Tap-A、Tap-B、Receiver Application 紀錄，並檢查時間同步、擷取完整性及接收端計數器。僅有 DUT／CI 摘要不足以放行。 |

### 二、單次測試觀測

| 量測位置 | 觀測結果 |
|---|---|
| Sender 發送紀錄 | **50,000** 個不同 `seq`，無重複送出；流量設定維持不變 |
| Tap-A（DUT 入口） | **50,000** 個不同 `seq`，無額外副本，`capture_drop=0` |
| Tap-B（DUT 出口） | **49,970** 個不同 `seq`，另有 **8** 個重複副本；`capture_drop=0` |
| Receiver Application | **49,900** 個不同 `seq`，另有 **8** 個重複副本；每個 `seq` 的首次接收時間已記錄 |
| Receiver Linux UDP 統計 | 本窗口 `RcvbufErrors` 增加 **70**；此主機只有本次 LAB 的 UDP 接收工作 |
| DUT Dashboard | `UDP packet loss = 0.06%`，`Interface drops = 0` |
| CI Smoke Test | `DUT_LOSS_BELOW_0_1=PASS`、`APP_RESPONSIVE=PASS`，合計 `2/2 PASS` |

### 三、延遲與稽核資料

Sender、Tap 與 Receiver 共用已校正的時間參考。應用程式端到端單向延遲的**擴展量測不確定度為 ±0.3 ms**。

| 來源／計算窗口 | p99 單向延遲 | 備註 |
|---|---:|---|
| Tap-A → Tap-B（線路） | **7.8 ms** | 以兩端皆擷取到的相同原始序號計算 |
| Sender → Receiver Application | **14.6 ms** | 以接收程式實際取得的 49,900 個不同序號計算 |

額外資料：Tap-A 與 Tap-B 的擷取設備自報 `capture_drop=0`，但尚未完成**跨 Tap 的序號集合與封包雜湊核對**；Receiver 應用程式使用的記錄程式已保存原始檔，尚未以獨立分析程式重算結果。

RD 表示：「DUT 只有 0.06% 丟包，Interface drops 又是 0，CI 全部 PASS。延遲在 Tap-B 也只有 7.8 ms；Receiver 的數字應該不是網路問題，可以先 Release。」

---

## (1) Evidence Analysis｜不同量測點看見的是同一件事嗎？

**A.** 根據現有證據，對 REQ-01～REQ-03 各標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，寫出**數據、分母與判準**。請區分「已觀測的規格符合性」、「量測仍需獨立核實」與「尚不能確認的唯一根因」。

**B.** 說明 `Tap-B 49,970 unique + 8 duplicates`、`Application 49,900 unique + 8 duplicates`、`RcvbufErrors +70`、`DUT Interface drops=0`、`CI 2/2 PASS` 各能證明什麼、不能證明什麼。為什麼不能直接把某個 Counter 當成整條路徑的遺失率？

**C.** Tap-A → Tap-B 的 `p99=7.8 ms` 與端到端 Application 的 `p99=14.6 ms` 是否在驗證**同一個需求終點**？對 `±0.3 ms` 的量測不確定度，你如何處理邊界判讀？

## (2) Verification Design｜最多兩個高鑑別度流程

你可以在 LAB 固定 Sender 速率與背景條件、重新建立 Socket、調整測試用 Receiver 的接收緩衝配置、取得多點 PCAP 和主機計數器、以獨立程式重新比對序號與時間戳；不得更改 REQ-01～REQ-03 的門檻、偷偷減低流量或將重複封包算成成功交付。

請設計**最多兩個**流程，區分三種假設：**H1** 傳輸路徑中確有封包遺失或延遲；**H2** Receiver 端 Socket／程式處理造成額外遺失或延遲；**H3** 擷取完整性、時間關聯或統計方式造成觀測偏差。

每個流程填寫：

```text
Verification Flow 1 / 2:
Initial State / Controlled Variables:
Traffic / Window / Run ID:
Measurement Points and seq Correlation:
Independent Oracle and Counters:
Procedure / Controlled Variation:
PASS / FAIL Criteria:
How Results Distinguish H1 / H2 / H3:
Evidence to Preserve:
```

## (3) Release Gate｜修正後數字漂亮，如何證明不是量測假象？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`；請根據**已核准需求**而不是 RD 對根因的推測作答。

**B.** 假設 RD 修改接收端設定後，Dashboard 顯示 `loss=0.02%`，Receiver Application 的摘要顯示 `loss=0.04%`、`p99=10.5 ms`。提出**兩項彼此獨立且可用同一 `run_id` 對照的回歸證據**，分別列出來源、相關欄位與 PASS／FAIL 判準。

**C.** 設計一個可安全復原的 **Negative Control（負向對照）**，讓測試流程在已知存在「Receiver 應用程式漏收序號，但 DUT Dashboard 仍顯示 PASS」的條件下，必須能回報失敗。說明注入或模擬方式、預期可觀測證據與復原方法。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪僅提供題目與必要名詞解釋，不附答案、提示性解答或解析；作答後再逐項批改證據判讀、測試設計與 Release Decision。
