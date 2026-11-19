# 今日技術練習｜2026/11/19

**主題：軟體測試／驗證策略 × MAC 封鎖 × IPv4/IPv6 × Release Gate**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜管理介面顯示「封鎖成功」，設備卻仍能連上 Internet

你負責驗證一台企業路由器／無線 AP 的 **MAC 位址 Internet 封鎖功能**。管理者將裝置 MAC 加入封鎖清單後，該裝置應失去 Internet 存取，但仍能使用區域網路（LAN）。產品支援 Router 與 AP/Bridge 模式，且要涵蓋有線、Wi-Fi、IPv4 與 IPv6。

本題是原創、合法隔離 LAB；所有位址、數據、時間及設備名稱均為虛構。所謂「Internet」是 LAB 內模擬的外部網段，沒有向公網發送測試封包。

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義 | 本題用途 |
|---|---|---|
| DUT | Device Under Test；受測設備 | 指提供 MAC 封鎖功能的路由器／AP。 |
| MAC | Media Access Control Address；媒體存取控制位址 | 用來在本題識別被指定封鎖的用戶端網路介面。 |
| ACL | Access Control List；存取控制清單 | 用來描述允許或拒絕特定流量的政策。 |
| Router／AP/Bridge | 路由器／無線基地台與橋接模式 | 本題的兩種獨立受測運作模式。 |
| LAN | Local Area Network；區域網路 | 本題指 DUT 下游的本地服務與測試主機。 |
| IPv4／IPv6 | Internet Protocol version 4／6；網際網路協定版本 | 本題須分別觀察兩種協定的封鎖結果。 |
| QUIC | 基於 UDP 的傳輸協定 | 在本題代表一條已建立、持續傳送應用資料的連線。 |
| Existing／New Flow | 既有／新建流量 | 分別指封鎖前已建立、或封鎖後才發起的通訊。 |
| Data Plane | 資料平面 | 指 DUT 真正轉送或阻止封包的行為。 |
| Tap／PCAP | Network Test Access Point／Packet Capture；分接點／封包擷取檔 | 用來獨立量測線路上實際經過的封包。 |
| Release Gate | 發布關卡 | 根據需求與驗證證據決定版本是否可放行。 |
| Negative Control | 負向對照 | 在隔離環境放入已知不符合需求的狀態，確認測試能察覺。 |

### 一、需求規格

封鎖生效時間以管理介面收到 DUT **成功儲存回覆的時刻 t0** 為起點；LAB 時間戳已校正，跨量測點的最大誤差為 ±0.2 秒。

| ID | Requirement |
|---|---|
| REQ-01 | 指定 MAC 加入封鎖清單後 **10.0 秒內**，該介面的 IPv4、IPv6 Internet 流量均不得再成功傳送應用資料；包含封鎖前已建立與封鎖後新建立的連線。 |
| REQ-02 | 封鎖不得阻止該裝置存取同一 LAN 上已核准的本地服務。 |
| REQ-03 | 功能須支援 Router、AP/Bridge 模式及 Wi-Fi、有線用戶端；封鎖清單在 DUT 重新開機後須保留且自動生效。 |
| REQ-04 | Release Gate 須用同一測試執行的 UI／政策狀態、DUT 入口封包、外部接收端資料、LAN 對照流量與重啟證據交叉驗證；僅有 UI 或 CI PASS 不足以放行。 |

註：每個 MAC 只代表**對應的網路介面**。本題 Wi-Fi 用戶端 A 與有線用戶端 B 各有獨立 MAC，不要求兩者共用識別碼。

### 二、測試拓撲

```text
Run R：Router Mode
Wi-Fi Client A (MAC 02:11:22:33:44:55)
      │
   [Tap-Down] ─ DUT (Router) ─ [Tap-Up] ─ LAB Internet Server
      │
      └── LAN Printer

Run B：AP/Bridge Mode
Wired Client B (MAC 02:66:77:88:99:AA)
      │
   [Tap-Down] ─ DUT (AP/Bridge) ─ [Tap-Up] ─ Upstream Gateway ─ LAB Internet Server
      │
      └── LAN File Server
```

兩個 Run 都先確認既有 IPv4 TCP 與 IPv6 QUIC 連線可正常收發，再透過 Web UI 加入各自的 MAC。外部 Server 的應用日誌與 Tap 封包可藉由測試專用的 `run_id`、來源／目的位址、連接埠及請求編號相互關聯。

### 三、測試結果

每個 Run 的時間獨立從各自的 `t0=0.0s` 計算。

| Run | 驗證項目 | 獨立觀測 |
|---|---|---|
| R：Router／Wi-Fi A | IPv4 既有 TCP | `t0+10s` 至 `t0+60s` 外部 Server **未收到**來自 A 的應用資料 |
| R：Router／Wi-Fi A | IPv6 既有 QUIC | 外部 Server 仍在 `t0+12.3s`、`+19.1s` 收到 A 的有效應用資料 |
| R：Router／Wi-Fi A | IPv6 新建連線 | `t0+15.1s`，外部 Server 完成一筆來自 A 的新請求 |
| R：Router／Wi-Fi A | LAN 印表機 | `t0+18.0s`，本地列印測試成功 |
| B：AP/Bridge／Wired B | IPv4 新建連線 | `t0+13.0s`，外部 Server 完成一筆來自 B 的新請求 |
| B：AP/Bridge／Wired B | IPv6 既有 QUIC | `t0+16.0s`，外部 Server 收到 B 的有效應用資料 |
| B：AP/Bridge／Wired B | LAN 檔案服務 | `t0+17.0s`，本地檔案讀取成功 |

補充紀錄：

```text
Run R:
  t0=14:00:00.000  Web UI: block MAC A -> saved
  t0+1.2s        DUT Dashboard: A status=BLOCKED

Run B:
  t0=15:00:00.000  Web UI: block MAC B -> saved
  t0+1.5s        DUT Dashboard: B status=BLOCKED

CI Smoke Test:
  UI_LIST_CONTAINS_MAC_A = PASS
  UI_LIST_CONTAINS_MAC_B = PASS
  ROUTER_IPV4_BROWSER_BLOCK = PASS
  LAN_R_PRINT = PASS
  LAN_B_READ = PASS
  DEVICE_HEALTHY = PASS
  Overall: 6/6 PASS
```

DUT 封鎖規則的完整有效配置、兩個 Run 的全量 PCAP 與重開機後結果**尚未取得**。上述外部 Server 事件已有獨立日誌，但目前尚未證明是哪一層或哪一條規則造成各項結果。

RD 認為：「兩個 MAC 都顯示 BLOCKED，CI 6/6 PASS；Router 模式的 IPv4 也擋住了，應該可以 Release。」

---

## (1) Evidence Analysis｜政策狀態與封包結果

**A.** 對 REQ-01～REQ-04 分別標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，說明你採用的證據。請區分**已觀測的結果**、**尚未驗證的需求範圍**、**尚未確認的根因**。

**B.** `BLOCKED` Dashboard、`6/6 PASS`、外部 Server 有效應用資料、LAN 服務成功，各能證明什麼、不能證明什麼？封鎖 Internet 時，為什麼需要分別看「既有」與「新建」連線？

## (2) Verification Design｜最多兩個高鑑別度測試流程

請設計**最多兩個可重複流程**，完整涵蓋 Router／Wi-Fi 與 AP/Bridge／有線、IPv4／IPv6、既有／新建 Internet 流量、LAN 不受影響及重開機後保留設定。

可在隔離 LAB 還原 DUT Configuration Snapshot，使用可控的外部 Server、Tap、封包產生器及受控重啟。不得放寬 10 秒要求，不得僅依 UI 或 DUT 計數器判定。

每個流程填寫：

```text
Test Flow ID:
Initial State / DUT Mode / MAC:
Controlled Variables / t0 Definition:
Traffic Matrix (IPv4, IPv6, Existing, New, LAN):
Measurement Points and Correlation Fields:
Procedure / Reboot Check:
Independent Oracle:
PASS / FAIL Criteria:
Evidence to Preserve:
```

## (3) Release Gate｜修正後如何證明真的擋住？

**A.** 請選擇 `Go`、`Hold` 或 `No-Go`，說明需求符合性與根因調查的差別。

**B.** 假設 RD 更新 Firmware，Dashboard 顯示兩個 MAC 均為 `BLOCKED`，CI 改成 `8/8 PASS`。請提出**兩組來源不同、能對應同一版本／Run 與時間窗口**的獨立回歸證據，各寫出來源、觀測內容與 PASS／FAIL 判準。

**C.** 請設計一個安全、可還原的 **Negative Control（負向對照）**，證明你的驗證機制能發現「管理介面顯示 BLOCKED，但 IPv6 既有連線仍可傳送應用資料」的情況。請寫出注入條件、預期偵測結果及還原方式。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪只出題並提供名詞解釋；待作答後才進行逐項批改、觀念解析與排查思路。
