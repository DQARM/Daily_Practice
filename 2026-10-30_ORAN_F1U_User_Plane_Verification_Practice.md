# 今日技術練習｜2026/10/30

**主題：5G／O-RAN × F1-U／NG-U × User-Plane Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜只出題，不附答案**

## Q1｜F1-C 正常、UE 也成功建立 PDU Session，為什麼 UL Throughput 只有 7.2 Mbps？

你負責驗證一套採用 O-RAN Split 7.2x、並將 gNB-CU 拆成 O-CU-CP 與 O-CU-UP 的 5G SA LAB。所有 IP、數據與門檻均為**本題虛構的教學情境設定**。

### 一、測試拓撲

```text
                         O-CU-CP
                        /       \
                    F1-C         E1
                      /           \
UE ⇄ O-RU ⇄ O-DU ── F1-U ── O-CU-UP ── NG-U (N3) ── UPF ── DN UDP Server
           （UL 資料路徑向右）
```

測試使用單一 UE、固定 RF 條件、固定 TDD Pattern，並在 60 秒內由 UE 向 DN UDP Server 傳送 **30 Mbps** 的上行 UDP 流量。每筆應用層 UDP Payload 為 **1,000 Bytes**。本次測試不預期有其他 UE 的 User-Plane 流量。

### 二、驗收要求

| ID | Requirement |
|---|---|
| REQ-01 | UL UDP Application Payload Throughput ≥ **27 Mbps**（60 秒平均）。 |
| REQ-02 | PDU Session 與相關 Bearer 在測試期間維持建立，不得非預期釋放。 |
| REQ-03 | Release 判斷必須有端到端量測與可追溯的分段證據，不能僅以 Control-Plane 成功訊息判定 User-Plane 正常。 |

### 三、測試證據

**Evidence A｜Control Plane**

```text
UE Registration                SUCCESS
PDU Session Establishment     SUCCESS
F1AP UE Context Setup         SUCCESS
E1 Bearer Context Setup       SUCCESS
Unexpected Session Release    0
```

**Evidence B｜同一個 60 秒測試窗口的流量觀測**

| 量測點 | 觀測值 |
|---|---:|
| UE Application UDP Sent | 225,000 datagrams |
| O-DU F1-U Egress Capture | 223,800 GTP-U packets |
| O-CU-UP F1-U Ingress Capture | 223,650 GTP-U packets |
| O-CU-UP NG-U Egress Capture | 54,200 GTP-U packets |
| DN UDP Server Received | 53,900 datagrams |
| DN 實測 UL Payload Throughput | 7.2 Mbps |
| O-RU／O-DU UL BLER（期間平均） | 0.4% |

**重要量測限制：** 兩處 GTP-U Capture 目前只依端點 IP 與 UDP Port 篩選，尚未完成 TEID、DRB／PDU Session 與原始 UDP Datagram 的逐筆關聯；各量測點也尚未確認 Capture Drop 與 Offload 影響。因此，表中計數**不能直接視為已完成同一批封包的端到端對帳**。

RD 的結論是：

> F1AP、E1 都成功，而且 O-DU 到 O-CU-UP 的封包數差不多，代表 F1-U 沒問題。吞吐量下降應該是 UPF 或 UDP Server 的問題。

---

## (1) Evidence Analysis｜控制面成功能證明使用者面正常嗎？

請回答：

**A.** Evidence A 能證明哪些狀態？是否足以證明 F1-U、NG-U 的 UL User-Plane 傳輸正確？

**B.** 針對 Evidence B，哪些數據可以直接比較，哪些比較還需要額外的封包識別或量測條件？請指出**至少兩個**證據缺口。

**C.** RD 將問題歸因於 UPF／UDP Server，你是否接受？請分成「已確認事實」「合理假設」「尚不能下的結論」三層回答。

## (2) Troubleshooting｜只允許兩個額外驗證動作

你需要區分三個假設：

| Hypothesis | Description |
|---|---|
| H1 | Radio／O-DU 上行資料在進入 F1-U 前已出現問題。 |
| H2 | F1-U Transport 或 O-CU-UP 的 Tunnel／Bearer 處理有問題。 |
| H3 | NG-U、UPF 或 DN 測試端造成吞吐量下降。 |

請設計**兩個資訊價值最高**的驗證動作；不可只回答「抓封包」或「看 Log」。每個動作都要包含：

```text
Verification 1 / 2
執行位置與工具／計數器：
要關聯的識別資訊或時間窗口：
預期觀察：
不同結果如何支持／排除 H1、H2、H3：
```

**限制：** 不改動 RF 條件、TDD Pattern 或 UDP Offered Load；不得直接假設 F1-U 與 NG-U 使用相同 TEID。

## (3) Release Gate｜尚未找到唯一根因，能不能判定規格？

目前 REQ-01 的量測結果為 **7.2 Mbps**，低於 **27 Mbps**；REQ-02 的 Control-Plane 證據如上；REQ-03 的逐段關聯尚未完成。

**A.** 請在 **Go／Hold／No-Go** 中選擇目前的 Release Decision，並說明「是否符合規格」與「是否找到根本原因」之間的關係。

**B.** 假設 RD 修正後，DN UDP Server 在單次 60 秒測試中量到 **28.5 Mbps**。請提出**最少兩項**你會要求補齊的 Verification Evidence，作為重新評估 Release 的依據，並說明各自的 PASS／FAIL 判準。

---

**作答方式：** 請在本題的對話分支中依序回答 (1)～(3)。首輪不提供答案；收到作答後，再解析核心原理、證據界線、常見誤區與排查策略。
