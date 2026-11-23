# 今日技術練習｜2026/11/23

**主題：5G／O-RAN × Inter-DU Handover × F1-U × End-to-End Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜UE 已完成切換，為什麼下行資料仍少了三筆？

你是 5G 系統驗證工程師，負責一個採用分離式 gNB 架構的隔離 LAB。UE 從 O-DU-A 的 Cell A 移動至 O-DU-B 的 Cell B；兩個 O-DU 共用同一組 O-CU-CP／O-CU-UP。研發團隊宣稱「RRC 切換完成，F1-U 上也看得到封包，所以 Handover PASS」。

本題拓撲、識別碼、時間、測試門檻與數據皆為**原創虛構 LAB 資料**，不是 3GPP／O-RAN 通用驗收限值。所有實驗只在授權隔離環境執行。

```text
                       O-CU-CP
                         │ F1-C
                    ┌────┴────┐
                    │         │
DL Generator ─ O-CU-UP ─ F1-U Switch ─┬─ [Tap-A] ─ O-DU-A ─ Cell A
                     │                └─ [Tap-B] ─ O-DU-B ─ Cell B
                  [Tap-CU]                         │
                                                    UE ─ UE App Log
```

`Tap-CU` 量測 O-CU-UP 的 F1-U 出口，`Tap-A`／`Tap-B` 分別量測兩個 O-DU 的 F1-U 入口。所有 Tap 使用已校正的硬體時間戳；**每個 Tap 的封包擷取完整性尚未完成獨立核對**。

### 名詞與縮寫速查

| 名詞 | 英文全名／繁體中文意義 | 本題用途 |
|---|---|---|
| UE／gNB | User Equipment／next generation NodeB；用戶設備／5G 基地台 | 分別代表接收端與無線接取網路節點。 |
| O-CU-CP／O-CU-UP | O-RAN Central Unit Control Plane／User Plane；中央單元控制平面／使用者平面 | 分別處理控制訊令及使用者資料。 |
| O-DU | O-RAN Distributed Unit；分散單元 | 在本題提供不同小區的無線接取功能。 |
| Inter-DU Handover | 跨分散單元切換 | UE 在本題從一個 O-DU 所屬小區切換到另一個 O-DU 所屬小區。 |
| RRC | Radio Resource Control；無線資源控制 | 提供 UE 無線連線的控制訊令。 |
| F1-C／F1AP | F1 Control Plane／F1 Application Protocol；F1 控制面／應用協定 | 用於 O-CU 與 O-DU 之間的控制程序。 |
| F1-U／GTP-U | F1 User Plane／GPRS Tunnelling Protocol User Plane；F1 使用者面／GPRS 通道使用者面協定 | 承載 O-CU-UP 與 O-DU 之間的使用者資料。 |
| TEID | Tunnel Endpoint Identifier；通道端點識別碼 | 用來辨識 GTP-U 通道端點；需搭配端點資訊關聯。 |
| PDCP SN | Packet Data Convergence Protocol Sequence Number；封包資料匯聚協定序號 | 用來關聯無線承載中的資料單元順序。 |
| `pkt_id`／`run_id` | 應用封包識別碼／測試執行識別碼 | 本 LAB 額外加入的關聯欄位，用來比對同次測試的資料。 |
| BLER | Block Error Rate；區塊錯誤率 | 表示無線鏈路傳輸區塊的錯誤比例。 |
| `capture_drop` | 擷取遺失計數 | 指量測設備未記錄到經過的封包，與線路實際丟包不同。 |
| Test Oracle／Release Gate | 測試判準依據／發布關卡 | 分別用來核對需求與決定版本是否符合發布條件。 |

### 一、已核准驗收需求

本 LAB 只驗證一個指定 UE、同一個下行資料承載及一次跨 DU 切換。測試產生器在 2 秒窗口內以 **每 1 ms 一筆**送出 `pkt_id=1～2000`，每筆含唯一 `pkt_id`。UE App 在發送窗口結束後額外等待 **500 ms** 收尾，再彙整結果。

| ID | Requirement |
|---|---|
| REQ-01 | UE 應從 Cell A 切換至 Cell B，完成 RRC 切換程序，維持同一個資料會話，且本次流程不得發生 RRC Re-establishment。 |
| REQ-02 | UE App 在指定收尾期結束前，至少須收到 **1999 個不同 `pkt_id`**，且不得將重複 `pkt_id` 交付給 App。 |
| REQ-03 | 以 UE App **相鄰兩筆實際交付資料**的接收時間計算，窗口內最大間隔不得超過 **30.0 ms**。 |
| REQ-04 | Release Gate 必須具備 RRC／F1AP 切換證據、CU-UP 與兩個 DU 方向的 F1-U 獨立封包觀測、`TEID`／`PDCP SN`／`pkt_id` 關聯、UE App 結果及擷取完整性證據；不得單憑 Handover Complete、儀表板或平均 BLER 放行。 |

### 二、控制面與應用層觀測

| 資料來源 | 觀測 |
|---|---|
| O-CU-CP Trace | `10:00:10.000` 開始切換控制程序；`10:00:10.046` 記錄目標小區的完成事件 |
| UE RRC 節錄 | `10:00:10.044` 記錄 `RRCReconfigurationComplete`；**完整 UE RRC 紀錄尚未封存** |
| UE Session Dashboard | 切換前後 `session_active=true`；未保存獨立的資料會話識別碼及相關事件紀錄 |
| DL Generator | `run_id=HO-1123-01`，確認送出 2000 個不同 `pkt_id`，無重送 |
| UE App 原始紀錄 | 收到 **1997 個不同 `pkt_id`**、無重複；缺少 `999、1001、1002` |
| UE App 接收時間 | 最大相鄰接收間隔 **46.0 ms**；擴展量測不確定度 **±1.0 ms** |
| O-DU-B Dashboard | 平均 DL BLER = `1.5%`；無逐筆 `pkt_id` 關聯 |
| CI | `HO_COMPLETE=PASS`、`SESSION_ACTIVE=PASS`，合計 `2/2 PASS` |

### 三、F1-U 封包節錄

O-CU-UP 的測試儀表已記錄本次資料承載的映射：來源路徑 `TEID=0xA100`、目標路徑 `TEID=0xB200`。`PDCP SN` 與 `pkt_id` 的對應由本 LAB 的 CU-UP Trace 提供，尚未以獨立方式核對。

| `pkt_id` | `PDCP SN` | `TEID` | Tap-CU | 對應 DU 入口 Tap | UE App |
|---:|---:|---|---|---|---|
| 997 | 3001 | `0xA100` | 有 | Tap-A：有 | 有 |
| 998 | 3002 | `0xA100` | 有 | Tap-A：有 | 有 |
| 999 | 3003 | `0xA100` | 有 | Tap-A：有 | **無** |
| 1000 | 3004 | `0xB200` | 有 | Tap-B：有 | 有 |
| 1001 | 3005 | `0xB200` | 有 | Tap-B：**未擷取到** | **無** |
| 1002 | 3006 | `0xB200` | 有 | Tap-B：有 | **無** |
| 1003 | 3007 | `0xB200` | 有 | Tap-B：有 | 有 |

其餘證據：三個 Tap 自報 `capture_drop=0`，但尚未完成跨 Tap 的封包數／雜湊完整性核對；O-DU-A／O-DU-B 的逐筆排程、HARQ 與 UE 無線接收 Trace 尚未取得。上表的「無」只代表該量測點**未觀察到**對應資料。

RD 認為：「RRC 已 Complete、會話仍 Active、F1-U 有流量，平均 BLER 只有 1.5%，所以這次切換可以 Release。」

---

## (1) Evidence Analysis｜控制面成功是否代表資料面符合要求？

**A.** 對 REQ-01～REQ-04 分別標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，逐項說明現有證據能支持的結論。請區分**實際觀測結果**、**尚需確認的量測可靠性**與**尚未確立的故障根因**。

**B.** `RRCReconfigurationComplete`、`session_active=true`、`Tap-CU 有封包`、`Tap-B 未擷取到 pkt_id=1001`、`BLER=1.5%`、`UE App 少三筆` 各能證明什麼、不能證明什麼？目前能否唯一歸因於 F1-U Switch？

**C.** 對 UE App 的 `46.0 ms ±1.0 ms` 與 REQ-03 的 `30.0 ms` 門檻，應如何記錄量測判定？請說明分母、時間點與對照資料的必要性。

## (2) Verification Design｜最多兩個高鑑別度流程

請在**同一個隔離 LAB**設計最多兩個流程，用以區分：H1＝F1-U 路徑有遺失／轉送異常；H2＝資料到達 O-DU 後，無線排程、切換緩衝或 UE 接收／上層交付異常；H3＝封包擷取、映射或時間關聯存在偏差。

可控制切換觸發時點、維持固定下行負載、封存 RRC/F1AP/UE Trace、取得 DU 無線排程與 HARQ 記錄、使用外部 Tap 與獨立封包關聯程式。不得放寬 REQ-01～REQ-04、降低負載或把 Dashboard 當成唯一判準。

每個流程填寫：

```text
Verification Flow 1 / 2:
Initial State / Controlled Variables:
Handover Trigger and Traffic Window:
Measurement Points / Correlation Fields:
Independent Oracle:
Procedure:
PASS / FAIL Criteria:
How It Distinguishes H1 / H2 / H3:
Evidence to Preserve:
```

## (3) Release Gate｜修正後還需要哪些獨立證據？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`。請區分「驗收結果」、「仍缺少的證據」與「尚未證實的唯一根因」。

**B.** 假設 RD 更新軟體後，Dashboard 顯示 `HO=SUCCESS`、`DL_loss=0%`，UE App 單次測試顯示 `2000/2000`。請提出**兩組來源不同、可關聯同一 `run_id`、版本與 UE** 的回歸證據，各列出來源、欄位與 PASS／FAIL 判準。

**C.** 設計一個安全可還原的 **Negative Control（負向對照）**，證明你的驗證流程能抓到「切換成功，但 UE App 漏收資料」的已知違規情況。說明受控故障／替代資料、預期偵測與還原方式。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪只提供題目與必要名詞解釋，不附答案、提示性解答或解析；收到作答後，再逐項批改證據判讀、測試設計與 Release Decision。
