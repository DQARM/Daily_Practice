# 今日技術練習｜2026/11/11

**主題：5G／O-RAN × eCPRI Fronthaul × Downlink Timing Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜PTP 顯示 LOCKED、交換器沒有 Drop，為什麼 O-RU 仍漏發下行符號？

你是系統驗證工程師，負責測試一套採用 **O-RAN split 7.2x** 的隔離 LAB。O-DU 經前傳（Fronthaul）交換器連接 O-RU。測試期間持續有背景流量，並以獨立 RF 分析儀觀測 O-RU 的下行訊號。所有門檻、封包、時間與結果均為本題虛構 LAB 資料，**不是通用的 O-RAN／3GPP 標準限值**。

```text
O-DU ── [Tap A] ── Fronthaul Switch ── [Tap B] ── O-RU ──(RF)── Analyzer
          │                                  │
          └──── 獨立硬體時間戳量測系統 ────────┘
                     （共同時間參考）
```

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義 | 在本題中的用途 |
|---|---|---|
| O-DU／O-RU | O-RAN Distributed Unit／Radio Unit；分散單元／無線單元 | 分別代表前傳兩端的設備。 |
| split 7.2x | O-RAN 功能分割方式 | 指 O-DU 與 O-RU 在部分實體層功能間分工。 |
| eCPRI | evolved Common Public Radio Interface；演進型通用公共無線介面 | 可用來承載前傳的資料訊息。 |
| C-plane／U-plane | Control Plane／User Plane；控制平面／使用者平面 | 本題前者提供傳輸配置資訊，後者承載 IQ 資料。 |
| IQ | In-phase / Quadrature；同相／正交分量 | 用來表示無線訊號的複數取樣資料。 |
| eAxC ID | extended Antenna-Carrier Identifier；延伸天線載波識別碼 | 用於區分前傳中的特定資料流。 |
| `sectionId`／`seqId` | Section Identifier／Sequence Identifier；區段識別碼／序號 | 用來辨識資料所屬的區段及封包順序；本題序號皆在同一個 8-bit 循環內。 |
| OTA／RF | Over-the-Air／Radio Frequency；空中傳送／射頻 | OTA 時點是預定無線發送時間；RF 分析儀量測實際無線輸出。 |
| PTP | Precision Time Protocol；精密時間協定 | 提供設備間的時間同步機制。 |
| Tap | Network Test Access Point；網路測試分接點 | 提供不依賴設備 Dashboard 的網路封包觀測位置。 |
| `capture drop` | 擷取遺失 | 指量測設備未成功記錄經過的封包，不一定代表網路丟包。 |

### 一、已核准的驗收要求

僅針對 **下行 eAxC `0x0021`、`sectionId=17`** 驗收；本題 LAB 已固定為每個受測下行 symbol 對應 **一個完整 U-plane 封包**，沒有分片，且已提供核准的 C-plane 排程表。

| ID | Requirement |
|---|---|
| REQ-01 | 在 120 秒穩態窗口內，每個應傳送的 U-plane 封包都必須在 O-RU 入口被收到，且相對於對應 OTA 時點的**接收時間介於 -80 μs 至 -25 μs（含端點）**；不允許遺失或超出窗口。 |
| REQ-02 | 同一 120 秒窗口內，獨立 RF 分析儀必須確認**所有排定的下行 symbol 都有實際輸出**，不得缺少。 |
| REQ-03 | 若發生 U-plane 封包遺失或逾越接收窗口，O-RU 必須在事件發生後 **2 秒內產生可稽核的 transport fault 告警**。 |
| REQ-04 | Release Gate 必須具備可追溯的 C-plane／U-plane 關聯、獨立前傳時間量測、RF 結果與事件時間關聯；不能只依 PTP LOCKED、設備 Dashboard 或交換器 Drop Counter 宣告 PASS。 |

### 二、120 秒測試窗口的觀測

測試窗口：`09:30:00`～`09:32:00`。Tap A 位於 O-DU 出口，Tap B 位於 O-RU 入口前的外部線路。兩個 Tap 的硬體時間戳與 OTA 參考已同步校正；**下表每筆 Tap B 接收時間相對 OTA 的擴展量測不確定度為 ±2 μs**。時間值為負代表比預定 OTA 時點早。

| 指標 | 觀測 |
|---|---|
| O-DU／O-RU PTP 狀態 | `LOCKED`／`LOCKED` |
| C-plane 排程 | Dashboard 顯示本窗口的排程完整送出 |
| Switch egress Drop Counter | `0` |
| O-RU Dashboard | `U-plane on-time = 99.98%` |
| O-RU underflow counter | 在本窗口增加 `2` |
| 外部 RF 分析儀 | 依核准排程應有的 `6,000` 個受測 symbol 中，確認 `5,998` 個有輸出、`2` 個未觀測到；儀器自我檢查顯示沒有 RF 擷取溢位 |
| Tap B 封包擷取 | 尚未完成擷取完整性／capture-drop 的獨立查核 |

QA 從同一個短時間片段抽出以下對應資料。四列的 eAxC、`sectionId`、方向均相同，且封包已依核准 C-plane 排程對應到各自的 OTA 時點。

| DL symbol | U-plane `seqId` | Tap A：O-DU 送出時間（相對 OTA） | Tap B：O-RU 入口時間（相對 OTA） |
|---|---:|---:|---:|
| S1 | 41 | -100 μs | -43 μs |
| S2 | 42 | -96 μs | **-14 μs** |
| S3 | 43 | -101 μs | **未觀察到** |
| S4 | 44 | -99 μs | -41 μs |

其他事件資料：

```text
09:31:14.xxx  RF Analyzer：記錄到缺少下行 symbol（細部時間戳可取回）
09:31:14.xxx  O-RU：underflow counter 增加（原始事件欄位尚未匯出）
09:31:18.300  NMS：收到 transport fault 告警
```

NMS 記錄的是**告警收到時間**。目前尚未取得 O-RU 告警**產生時間**、事件的 `seqId`／symbol 關聯欄位，也尚未完成 O-RU／NMS Log 時鐘的偏差校正。

RD 認為：「PTP LOCKED、Switch Drop=0，U-plane on-time 也有 99.98%，因此網路沒問題。RF 少兩個 symbol 可能是儀器造成；NMS 收到告警就算有告警，可以放行。」

---

## (1) Evidence Analysis｜同步、封包到達與實際發射

**A.** 依現有證據，對 REQ-01～REQ-04 分別標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，逐項說明依據。對 `S2` 請明確說明如何考慮 **±2 μs 量測不確定度**；對 `S3` 則區分「未擷取到」與「已證實網路丟包」。

**B.** `PTP LOCKED`、`Switch Drop=0`、`U-plane on-time=99.98%`、`underflow +2`、`RF 少 2 個 symbol` 各能支持什麼、不能證明什麼？目前是否足以把 RF 缺失**唯一歸因**於前傳交換器？

**C.** 根據 `09:31:14.xxx` 的事件與 `09:31:18.300` 的 NMS 收到時間，能否直接斷定 REQ-03 違規？請列出仍缺少的時間與事件關聯證據。

## (2) Verification Design｜只增加兩個高鑑別度驗證流程

請在隔離 LAB 設計**最多兩個流程**，區分下列假設：

| 假設 | 描述 |
|---|---|
| H1 | 前傳網路確實造成 U-plane 封包遺失或抵達過晚。 |
| H2 | 封包擷取、時間對齊或 RF 量測造成觀測偏差。 |
| H3 | 封包到達後，O-RU 的內部處理、排程關聯或告警流程存在問題。 |

每個流程填寫：

```text
Verification Flow 1 / 2:
Controlled Variables / Initial State:
Tap / Device / RF Measurement Points:
Correlation Fields (eAxC, section, seq, symbol, timestamps):
Procedure and Fault Injection (if any):
Independent Oracle:
PASS / FAIL Criteria:
How Results Distinguish H1 / H2 / H3:
```

限制：不可更改既有驗收門檻、降低背景負載或直接把 Dashboard 當成真值。若設計故障注入，須說明如何證明**故障確實發生在指定位置與時間**，而不只是注入命令執行成功。

## (3) Release Gate｜如何避免「99.98% 就算很好」？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`；請區分「規格符合性」、「尚待核實的量測」與「尚未查明的根因」。

**B.** 假設 RD 修正後，Dashboard 顯示 `100% on-time`，RF 分析儀在單次 120 秒測試中觀察到 `6,000/6,000` 個 symbol。你還要求哪**兩項彼此獨立**的回歸證據，才能重新評估 REQ-01～REQ-04？請寫出來源、關聯欄位與 PASS／FAIL 判準。

**C.** 提出一個安全、可復原的 **negative control（負向對照）**，用來證明新增驗證流程確實能偵測「U-plane 封包超過到達時限」，而不只是無論資料如何都顯示 PASS。請列出實施條件、預期測試偵測行為與回復方式。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。本次僅提供題目與必要名詞說明，不附答案、提示性解答或解析；作答後再逐項批改證據判讀、測試設計與 Release Decision。
