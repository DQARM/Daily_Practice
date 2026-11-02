# 每日技術練習｜2026-10-28

**主題：Network / Performance × TCP MSS × Wire Rate × Verification Evidence**  
**難度：★★★★☆｜預估 10–15 分鐘｜僅出題，不附答案**

## 情境｜TCP Throughput 下降，是 DUT 效能不足，還是封包開銷？

你是 QA，負責驗證一台 1 GbE Network Appliance（DUT）的 IPv4/TCP Forwarding Performance。測試在隔離 LAB 進行，使用 `iperf3`，並以獨立 Network TAP 量測實際線速與封包數。**既有路徑 RTT 固定為 5 ms，不允許任意修改。**

```text
iperf3 Client ── 1 GbE ── DUT ── 1 GbE ── iperf3 Server
                                  │
                            Network TAP
RTT = 5 ms（固定）｜Full Duplex｜無背景流量
```

### 驗收規格（本 LAB 自訂）

- **REQ-01：** TCP 單連線、MSS = 1460 bytes，Application Payload Throughput ≥ 900 Mbps。
- **REQ-02：** TCP 單連線、MSS = 536 bytes，Application Payload Throughput ≥ 800 Mbps。
- **REQ-03：** 每次測試 60 秒，TCP Retransmissions < 0.1%。
- 測試條件：IPv4、Ethernet II、無 VLAN、無 IP/TCP Options；Ethernet MTU = 1500 bytes。
- 計算線上傳輸量時，**包含** Ethernet Header 14 B、FCS 4 B、Preamble/SFD 8 B、Inter-Frame Gap 12 B、IPv4 Header 20 B、TCP Header 20 B；不考慮 ACK 流量對反向鏈路的影響。
- `iperf3` Throughput 僅計 TCP Application Payload，不包含任何上述 Header / FCS / L1 開銷。

### QA 測試資料

| Measurement | Test A | Test B |
|---|---:|---:|
| TCP Connections | 1 | 1 |
| MSS | 1460 B | 536 B |
| RTT | 5 ms | 5 ms |
| Duration | 60 s | 60 s |
| iperf3 Payload Throughput | 940 Mbps | 760 Mbps |
| TAP Measured Wire Rate | 約 990 Mbps | 約 871 Mbps |
| TCP Retransmissions | 0 | 0 |
| Effective RWND（穩態） | ≥ 4 MiB | ≥ 4 MiB |
| Sender CWND（穩態） | ≥ 2 MiB | ≥ 2 MiB |
| DUT CPU（整體平均） | 38% | 42% |
| Server 某個 CPU Core 峰值 | 64% | 92% |
| DUT Interface Drops | 0 | 0 |

額外執行 **Bypass Reference Test**：保留同一組 Client / Server、MSS = 536 B、RTT = 5 ms，改用已驗證的參考轉送路徑繞過 DUT，量得 **845 Mbps** Application Payload Throughput、0 次 Retransmission。這個參考結果不是 DUT 的測試結果。

RD 說：「Test B 因為小封包 Header 比例比較高，所以 760 Mbps 是正常的；DUT 沒有 Drops，也沒有 Retransmission，應該算 PASS。」

---

## (1) Evidence Analysis｜請先計算，再判斷

**A.** 依題目提供的 Header / FCS / L1 開銷，計算 Test A 與 Test B **每個滿載 TCP Data Segment 的 On-Wire Bytes**，再計算在 **1 Gbps 實體線速**下各自的理論最大 **TCP Payload Throughput（Mbps）**。請列出公式與單位。

**B.** 根據 `iperf3` Throughput 與 MSS，估算 Test A、Test B 的 **TCP Data Segments per Second（pps）**。兩者的每秒封包處理需求有何差異？

**C.** RD 的說法有哪些部分能被目前證據支持？哪些仍是推測？請分別評估 `Wire Rate`、`Retransmissions = 0`、`DUT CPU 平均值`、`Server 單核心峰值` 與 `Bypass Reference Test` 的證據力。

## (2) Troubleshooting｜只能再做兩個驗證動作

請從以下假設中設計**兩個最有資訊價值的驗證動作**，**維持 RTT = 5 ms**，不得只回答「看 Log」或「重新跑一次」。

| Hypothesis | Description |
|---|---|
| H1 | 小 MSS 增加每秒封包處理負擔，DUT 轉送能力受到限制 |
| H2 | Client / Server 的 CPU、Socket 或封包處理路徑限制了測試結果 |
| H3 | 測試量測點、計數器口徑或 Offload 行為造成錯誤判讀 |

每個驗證動作請填寫：

```text
Test Objective:
Controlled Variables:
Measurement Points / Tools:
Expected Observations:
不同結果如何支持或排除 H1、H2、H3：
```

## (3) Release Gate｜目前可以放行嗎？

請從以下選擇一項，並說明**目前能確定的規格符合性**與**尚未證明的根本原因**有何差別。

- **A. Go：** Test A 達標，Test B 沒有 Retransmissions / Drops，且小封包原本就有額外開銷。
- **B. Hold：** Test B 未達標，但一定要先證明唯一 Root Cause，才能判斷是否符合規格。
- **C. No-Go：** Test B 未達明確的驗收門檻；即使根本原因尚未確認，也不能直接宣告符合 REQ-02。

**額外追問：** 如果 RD 修正後 Test B 達到 **815 Mbps**，你會要求補上哪**一項最關鍵的獨立驗證證據**，才接受 REQ-02 的 PASS 結論？請說明判準。

---

**作答原則：** 區分已知事實、計算推導、合理假設與未證明結論。此題先不提供答案；在本題分支作答後，再逐項解析。
