# 今日技術練習｜2026/11/13

**主題：軟體測試／驗證策略 × Fault Injection × Crash Consistency × Release Gate**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜設備重開機後顯示新版本，實際防火牆規則卻變成第三種狀態

你負責驗證一台企業網路閘道設備的防火牆政策更新功能。管理服務會將核准的政策檔部署到設備，更新過程可能遇到突然斷電。你需要判斷：**管理介面顯示更新完成，是否足以證明實際轉送行為正確？**

以下政策、位址、數據與紀錄全部為虛構、合法隔離 LAB 的設定。測試封包不會送到公網。已確認本 LAB 沒有其他政策來源、旁路或其他同時進行的設定變更。

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義 | 本題中的用途 |
|---|---|---|
| Control Plane／Data Plane | 控制平面／資料平面 | 前者管理政策設定，後者實際處理封包；可分開觀測。 |
| Atomic Update | 原子性更新 | 指一次更新對外呈現完整的前版或後版狀態，而非混合狀態。 |
| Crash Consistency | 當機一致性 | 指突然中斷後，持久化資料與恢復狀態需符合系統承諾。 |
| Fault Injection | 故障注入 | 在受控環境中刻意引入斷電、程序終止等事件。 |
| PDU | Power Distribution Unit；電源分配裝置 | 在本題用來執行並記錄受控斷電。 |
| SHA-256／Digest | Secure Hash Algorithm 256-bit／摘要值 | 用來比對政策檔內容是否與核准版本相符。 |
| HTTP `202` | Accepted；已接受處理 | 表示請求被接收，未必表示後續工作已完成。 |
| Commit Marker | 提交標記 | 用於辨識更新流程的持久提交狀態；其實際保存位置需查證。 |
| `nftables` | Linux 封包過濾框架 | 可用來檢視設備執行中的封包處理規則。 |
| Negative Control | 負向對照 | 人為放入已知違規條件，檢查測試能否偵測錯誤。 |

### 一、已核准的需求與政策

| ID | Requirement |
|---|---|
| REQ-01 | 只有經核准且簽章／SHA-256 驗證通過的完整政策版本，才允許成為設備生效政策。 |
| REQ-02 | 政策更新必須具備原子性；任何更新階段突然斷電後，恢復的**實際封包行為**必須完全符合舊版 `P20` 或新版 `P21`，不得出現混合或額外放行規則。 |
| REQ-03 | 對指定測試來源及目的地，管理埠 `TCP/8443` 在 `P20`、`P21` 與更新／恢復過程中都必須拒絕轉送。 |
| REQ-04 | Release Gate 必須包含獨立政策檔判準、設備重開機後的持久化證據、實際封包測試及時間關聯；不能只依管理 API、服務狀態或 CI PASS 放行。 |

所有測試封包均由 `192.0.2.10` 發送至 `198.51.100.20`，且以外部接收端是否實際收到封包為觀測依據。以下是**完整的受測流量決策表**：

| 政策版本 | TCP/443 | TCP/8443 |
|---|---|---|
| `P20`（舊版） | DENY | DENY |
| `P21`（核准新版） | ALLOW | DENY |

核准的唯讀政策儲存庫保存 `P20` 與 `P21` 的簽章、政策內容及對應 SHA-256，分別記為 `H20`、`H21`。`H20`、`H21` 是本題用於識別不同完整摘要的代號，**不是實際 64 位十六進位摘要**。

### 二、測試執行與獨立觀測

設備測試前正在執行 `P20`。QA 嘗試更新成 `P21`，並在更新流程中進行斷電注入。

```text
10:00:00.000  POST /policy/deploy P21 -> HTTP 202
10:00:02.180  Device log: phase=WRITING_RULES
10:00:02.300  PDU: power-off command issued
10:00:02.320  PDU power telemetry: output voltage drops to 0
10:00:06.xxx  Device restarts
10:00:12.000  Management API: active_policy=P21, status=READY
10:00:12.100  systemctl is-active policy-agent -> active
10:00:13.000  Offline read of active-policy.bin: SHA-256=H20
```

重開機後，QA 用**獨立封包產生器與接收端**進行各 20 次連線測試：

| 測試流量 | 接收端實際收到／送出 | DUT 執行中規則摘要 |
|---|---:|---|
| `TCP/443` | `20/20` | 有 `accept tcp dport 443` |
| `TCP/8443` | `20/20` | 有 `accept tcp dport 8443` |

其他證據：

| Evidence | Observation |
|---|---|
| CI Smoke Test | 管理 API `READY`、policy-agent `active`、可讀到版本 `P21`：`3/3 PASS` |
| 簽章驗證 | 管理 API 回報 `signature_ok=true`；尚無更新後的離線獨立驗證 |
| 持久提交紀錄 | 尚未取得交易／Commit Marker 的原始紀錄 |
| 故障精確時點 | 確認斷電發生，但尚未證實它在寫入、提交的哪個原子邊界生效 |
| 封包觀測 | 發送與接收端的測試編號及時間戳均已保存 |

RD 表示：「`P21` 顯示 READY，CI 也全部 PASS。`443` 已經能通，代表新版政策生效；`8443` 可能只是斷電後的暫時現象，先 Release 再追。」

---

## (1) Evidence Analysis｜管理狀態與實際行為

**A.** 對 REQ-01～REQ-04 各標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，逐項寫出依據。請區分**已觀測的規格符合性**與**尚未查明的故障根因**。

**B.** `HTTP 202`、`active_policy=P21`、`SHA-256=H20`、`3/3 PASS`、外部 `TCP/443` 與 `TCP/8443` 的結果，各能證明什麼、不能證明什麼？請區分 Control Plane、持久化儲存及 Data Plane。

**C.** 目前能否確定斷電是造成異常的唯一根因？若要確認更新在何種階段失去一致性，還需要哪些具體的版本、交易時點或開機還原證據？

## (2) Test Design｜只能新增兩個故障注入流程

你可在隔離 LAB 還原設備 Snapshot、控制 PDU、取得外部封包紀錄、離線比對政策檔，以及讀取設備開機／交易 Log；不得修改產品原始碼或放寬政策。請設計**最多兩個流程**，涵蓋更新提交邊界兩側的中斷與恢復行為。

每個流程填寫：

```text
Test Flow ID:
Initial Policy / Snapshot:
Fault Injection Point and Proof of Timing:
Procedure / Reboot Sequence:
Independent Policy and Data-plane Oracles:
Expected Persistent State / Packet Behavior:
PASS / FAIL Criteria:
Evidence to Preserve:
```

額外回答：如果 PDU 確實斷電，但無法判定當時 Commit Marker 是否已持久提交，你應如何設定可驗證、又不會錯誤假定提交時點的 PASS／FAIL 判準？

## (3) Release Gate｜修正後畫面全綠是否足夠？

**A.** 依目前證據選擇 `Go`、`Hold` 或 `No-Go`，並對 REQ-01～REQ-04 說明。不得把「懷疑斷電造成」寫成已確認的唯一根因。

**B.** 假設 RD 修正後，同一注入情境下管理 API 顯示 `P20`，外部量測顯示 `443=DENY`、`8443=DENY`。請提出**兩項彼此獨立、可關聯同一次重開機**的額外證據，寫明來源、關聯欄位與 PASS／FAIL 判準，作為重新評估 Release 的依據。

**C.** 請設計一個安全可還原的 **negative control（負向對照）**，驗證新增的測試流程確實能攔下「管理介面顯示正常，但實際 Data Plane 額外放行 `TCP/8443`」的版本。說明測試環境、已知違規條件、預期偵測結果與還原方式。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪只提供題目及必要名詞解釋，不附答案、提示或解析；作答後再逐項批改證據判讀、驗證設計、常見誤區與排查思路。

📄 **[下載 2026-11-13_Software_Config_Crash_Consistency_Practice.md](sandbox:/mnt/data/2026-11-13_Software_Config_Crash_Consistency_Practice.md)**
