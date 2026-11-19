# 每日技術練習｜2026/11/17

**主題：5G／O-RAN × F1／E1／NG 介面 × User Plane Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜PDU Session 建立成功，為什麼 UE 上行資料沒有到達 UPF？

你負責驗證一套合法隔離 LAB 的 5G SA 系統。UE 經 O-RU、O-DU、O-CU-UP 與 UPF 連到測試伺服器；O-CU-CP 負責控制面。所有數值與識別碼均為虛構測試資料。

```text
UE ⇄ O-RU ⇄ O-DU ── F1-U ── O-CU-UP ── N3 ── UPF ── DN Server
                 ╲ F1-C       │
                   O-CU-CP ─ E1
                      │
                    N2 / AMF
```

### 名詞與縮寫速查

| 名詞 | 英文全名／中文說明 | 本題用途 |
|---|---|---|
| UE | User Equipment；使用者設備 | 發送測試上行流量的終端。 |
| O-DU／O-CU-CP／O-CU-UP | Distributed Unit／Central Unit Control Plane／Central Unit User Plane；分散單元／集中控制面／集中使用者面 | 分別處理無線接取與控制、資料功能。 |
| F1-C／F1-U | F1 Control／User Plane；F1 控制／使用者介面 | 連接 O-DU 與 O-CU 的不同資料路徑。 |
| E1／E1AP | E1 Interface／E1 Application Protocol；集中單元內控制面與使用者面之間的介面及協定 | 傳遞 Bearer 控制相關資訊。 |
| N2／N3 | 5GC 控制面／使用者面介面 | N2 連 AMF，N3 連 UPF。 |
| NGAP／F1AP | NG／F1 Application Protocol；控制面應用協定 | 用來傳送特定控制面訊息。 |
| GTP-U／TEID | GPRS Tunnelling Protocol User Plane／Tunnel Endpoint Identifier；使用者面隧道協定／隧道端點識別碼 | 用於關聯不同介面上的使用者面封包。 |
| DRB | Data Radio Bearer；資料無線承載 | 關聯 UE 的使用者面資料承載。 |
| PDU Session | Protocol Data Unit Session；封包資料連線 | UE 與資料網路之間的邏輯連線。 |
| UL／DL | Uplink／Downlink；上行／下行 | 表示 UE 傳輸資料的方向。 |
| Capture Drop | 封包擷取遺失 | 擷取工具漏記封包的情況，不等同於網路真正丟包。 |

### 一、驗收需求

| ID | Requirement |
|---|---|
| REQ-01 | UE 必須完成 Registration 與 PDU Session Establishment，且測試期間連線不得非預期釋放。 |
| REQ-02 | 單 UE 上行 UDP Application Payload Throughput 在 60 秒窗口內須 ≥ 25 Mbps；提供 30 Mbps Offered Load。 |
| REQ-03 | 驗收必須能以同一 UE／DRB／PDU Session 的證據關聯 F1-U、N3 與 DN 收包，不得僅以控制面 Success 判定使用者面正常。 |

### 二、測試觀測

固定 RF、TDD Pattern 與單 UE；每個 UDP Datagram 的應用層 Payload 為 1,000 Bytes。測試發送 225,000 個 Datagram（60 秒），無其他使用者面流量。

| 量測點 | 同一窗口觀測 |
|---|---:|
| UE Registration | SUCCESS |
| PDU Session Establishment | SUCCESS |
| F1AP UE Context Setup | SUCCESS |
| E1AP Bearer Context Setup | SUCCESS |
| Unexpected PDU Session Release | 0 |
| UE App UDP Sent | 225,000 datagrams |
| O-DU F1-U egress | 224,200 GTP-U packets |
| O-CU-UP F1-U ingress | 224,050 GTP-U packets |
| O-CU-UP N3 egress | 61,000 GTP-U packets |
| DN UDP Server received | 60,500 datagrams |
| DN UL Payload Throughput | 8.07 Mbps |
| UL PUSCH CRC PASS | 99.6% |

目前 F1-U／N3 擷取僅以 IP、UDP 2152 過濾，**尚未**核對 TEID、QFI、DRB、內層 UE IP／UDP 序號；Capture Drop 與網卡 Offload 的影響也尚未查明。兩個介面的 TEID 不預設相同。

RD 表示：「Registration、F1AP、E1AP 都成功，F1-U 的進出數量也很接近，因此問題一定在 UPF。」

## (1) Evidence Analysis

**A.** REQ-01～REQ-03 分別應標記 `PASS`、`FAIL` 或 `NOT VERIFIED`？請區分已觀測的規格符合性與尚未確定的根本原因。

**B.** 控制面 Success、PUSCH CRC PASS、F1-U GTP-U 數量各能證明什麼、不能證明什麼？為何不同介面的 GTP-U 封包數不能在缺少關聯資訊時直接視為同一批流量？

**C.** RD 的「一定是 UPF」是否成立？請列出至少兩個競爭假設及目前欠缺的證據。

## (2) Verification Design

只能新增**兩個**驗證動作，區分：H1 O-DU／F1-U 轉送問題、H2 O-CU-UP Bearer／Tunnel Mapping 問題、H3 N3／UPF／DN 問題。不得改變 RF、Offered Load 或 TDD Pattern。

每個動作請填寫：

```text
Verification 1 / 2:
Measurement Points and Tools:
UE / DRB / PDU Session / TEID Correlation:
Controlled Variables and Time Window:
Expected Evidence:
How Results Distinguish H1 / H2 / H3:
PASS / FAIL Criteria:
```

## (3) Release Gate

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`，說明規格符合性與根因調查的區別。

**B.** 假設修正後 DN Server 收到 210,000 個 1,000-Byte UDP Payload Datagram（60 秒），請計算 Throughput，並提出**兩項互相獨立且可追溯**的補充證據，作為重新評估 REQ-02、REQ-03 的依據。

**C.** 若擷取工具報告 Capture Drop，如何避免將量測遺失誤判為真正的 User Plane Packet Loss？提出一項可執行的替代驗證方式。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。本次只提供題目與名詞解釋，不附答案、提示性解答或解析。
