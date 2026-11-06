# 今日技術練習｜2026/11/03

**主題：Network / Performance × TCP Retransmission × Independent Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜iperf3 顯示 920 Mbps，為什麼使用者下載卻只有 610 Mbps？

你負責驗證隔離 LAB 中一台 1 GbE Firewall DUT 的 TCP Forwarding Performance。所有數值與門檻均為本題自訂的測試資料。Client 與 Server 之間只有 DUT，沒有其他背景流量。

```text
Client (10.10.1.10) ── 1 GbE ── DUT ── 1 GbE ── Server (10.10.2.20)
                          │
                     Mirror Port → Packet Capture
```

### 一、驗收要求

| ID | Requirement |
|---|---|
| REQ-01 | 單一 TCP 連線、60 秒、RTT 20 ms，Application Payload Throughput ≥ 850 Mbps。 |
| REQ-02 | 同一測試窗口的 TCP Retransmitted Data Segments / Original Data Segments < 0.5%。此比例必須以可稽核的封包或端點計數證據驗證。 |
| REQ-03 | Release 判斷需包含獨立量測證據，不能只依賴 DUT Dashboard 或單一測試工具摘要。 |

### 二、測試結果

兩組測試使用相同 Client、Server、MTU 1500、RTT 20 ms、TCP 單連線、60 秒。`iperf3` 使用 `-P 1`；HTTP 測試下載同一份預先放在 Server 記憶體中的 8 GiB 測試檔，排除磁碟讀取限制的已知影響。

| Evidence | Test A：iperf3 | Test B：HTTP Download |
|---|---:|---:|
| Application Payload Throughput | 920 Mbps | 610 Mbps |
| Sender TCP Retransmissions（工具摘要） | 18 | 12,400 |
| Client TCP Receive Window（穩態） | ≥ 4 MiB | ≥ 4 MiB |
| Server TCP Congestion Window（穩態） | ≥ 2 MiB | 變動，最低 96 KiB |
| DUT CPU 平均 | 47% | 49% |
| DUT Interface Drops | 0 | 0 |
| HTTP 5xx Errors | N/A | 0 |

**補充證據：**

```text
Test B: Mirror Capture
  Filter: host 10.10.1.10 and host 10.10.2.20
  Captured packets: 4,120,000
  Capture tool reported dropped packets: 85,000
  TCP analysis: 10,900 packets marked "TCP Retransmission"

Test B: DUT Dashboard
  Forwarded TCP bytes: 4.7 GiB
  Policy action: ALLOW
```

目前尚未取得 Test B 的 Server-side TCP 計數器、Client-side Capture，以及 Mirror Port 是否因 Oversubscription 丟包的驗證結果。工具摘要中的 `12,400` 是重傳次數，不是已計算完成的重傳比例。

RD 認為：「iperf3 已達 920 Mbps、DUT CPU 不高、Interface Drops 為 0，所以 Firewall 效能 PASS；HTTP 比較慢應該是應用程式問題。」

## (1) Evidence Analysis｜哪些結論已成立？

**A.** 依 REQ-01 的文字與 Test A、Test B 的結果，你如何判斷規格符合性？請明確說明測試工作負載的差異是否影響判定，不能自行修改 REQ-01 的測試條件。

**B.** `iperf3 Retransmissions = 18`、`HTTP Retransmissions = 12,400`、Mirror Capture 標記 `10,900`，能否直接計算 REQ-02 的重傳比例？請指出缺少的分母、統計口徑或證據品質問題。

**C.** 對「DUT CPU 49%」「Interface Drops = 0」「HTTP 5xx = 0」「Mirror Capture dropped packets = 85,000」四項證據，逐項說明它們能證明什麼、不能證明什麼。

## (2) Troubleshooting｜只能新增兩個驗證動作

請在以下假設之間設計**兩個最有鑑別度的驗證動作**：

| Hypothesis | Description |
|---|---|
| H1 | DUT 或中間轉送路徑造成資料遺失／重傳。 |
| H2 | HTTP Server／Client 的 TCP 或應用層行為限制吞吐量。 |
| H3 | Mirror Capture 丟包、重傳辨識或工具統計口徑造成誤判。 |

每個動作必須包含：

```text
Verification 1 / 2:
Controlled Variables:
Measurement Points / Commands or Counters:
Expected Observations:
How Results Distinguish H1 / H2 / H3:
```

限制：不得變更 RTT、鏈路速率、DUT Policy 或測試檔案；不能只回答「抓封包」或「看 Log」。

## (3) Release Gate｜如何避免以單一漂亮數字放行？

**A.** 請選擇目前的 `Go`、`Hold` 或 `No-Go`，分別說明 REQ-01、REQ-02、REQ-03 的證據狀態。若你認為需求文字對 HTTP 是否納入驗收有歧義，請指出歧義與處理方式，不得擅自假設已釐清。

**B.** 假設 RD 修正後，HTTP Download 提升至 **875 Mbps**，但只有 DUT Dashboard 顯示成功。請提出**兩項互相獨立且可追溯到同一 60 秒窗口**的驗證證據，並為每項寫出 PASS／FAIL 判準。

**C.** 如果封包擷取設備本身持續丟包，你會如何處理 REQ-02 的驗收狀態？請提出一個可替代的獨立證據取得方案。

---

**作答方式：** 請在本題對話分支依序回答 (1)～(3)。本次不附答案或解析；收到作答後，再逐項檢查證據判讀、驗證設計與 Release Decision。
