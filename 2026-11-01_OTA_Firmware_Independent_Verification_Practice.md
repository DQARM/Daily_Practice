# 今日技術練習｜2026/11/01

**主題：軟體測試／驗證策略 × OTA Firmware × Independent Test Oracle**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜OTA 更新顯示 SUCCESS，設備卻仍執行舊版韌體

你負責驗證一款工業 IoT Gateway 的 OTA 韌體更新功能。設備採用 A/B 雙分割區：目前從 Slot A 執行 `v2.4.0`，預定更新 Slot B 至 `v2.4.1`。所有數值、路徑、日誌及門檻都是**本題虛構 LAB 設定**。

### 一、驗收需求

| ID | Requirement |
|---|---|
| REQ-01 | 更新成功並重新開機後，設備**實際啟動並執行** `v2.4.1`；執行中韌體的 SHA-256 必須與經核准的 Release Manifest 一致。 |
| REQ-02 | 在 Slot B 寫入途中斷電，設備重新上電後須能啟動可用的 `v2.4.0`，且原有網路設定不得遺失。 |
| REQ-03 | 若切換 Slot B 後健康檢查持續失敗，設備須於 90 秒內回復至可用的 Slot A，且保留原有網路設定。 |
| REQ-04 | Release Gate 必須以設備端可追溯的實際執行狀態與獨立驗證證據判定，不得只依賴更新服務回報。 |

### 二、既有測試與觀測

QA 執行正常更新測試，未主動斷電。測試工具從 OTA Service 取得結果：

```text
Package signature verification : PASS
Package SHA-256 vs manifest     : MATCH
POST /ota/update               : HTTP 202 Accepted
OTA Agent: write Slot B        : SUCCESS
OTA Agent: activate Slot B     : SUCCESS
OTA Agent: update completed    : SUCCESS
CI Pipeline                    : GREEN
```

重新開機後，QA 另外取得下列證據：

| 證據來源 | 觀測結果 |
|---|---|
| OTA Dashboard | `active_slot=B`, `version=2.4.1` |
| Bootloader boot record（本次開機） | `booted_slot=A` |
| Device runtime API `/system/version` | `2.4.0` |
| Device health endpoint | `HEALTHY` |
| 執行中 RootFS 的 SHA-256 | **尚未量測** |
| 網路設定持久化比對 | **尚未執行** |
| 斷電與健康檢查失敗測試 | **尚未執行** |

RD 表示：「更新流程全部成功，健康狀態也正常；Dashboard 的版本已經是 2.4.1，應該可以放行。Bootloader 記錄可能只是舊資料。」

---

## (1) Evidence Analysis｜哪一個 SUCCESS 具有什麼證明力？

**A.** 請分別說明 `HTTP 202`、`OTA Agent SUCCESS`、`Dashboard active_slot=B`、`Bootloader booted_slot=A`、`Runtime API=2.4.0`、`HEALTHY` 各自**能證明什麼、不能證明什麼**。

**B.** 依現有證據，REQ-01～REQ-04 各自應標記為 `PASS`、`FAIL` 或 `NOT VERIFIED`？請逐項說明，並區分**觀測到的規格問題**與**尚未確定的根本原因**。

## (2) Test Design｜有限時間下設計兩個高價值測試

你可以控制 LAB 電源、觸發健康檢查失敗、讀取 Bootloader 記錄與執行中 RootFS、存取獨立保存的 Release Manifest，但不能修改產品原始碼。

請設計**兩個測試流程**，盡可能涵蓋正常更新、寫入中斷電、切換後健康檢查失敗三種風險。每個流程請填寫：

```text
Test Objective:
Preconditions / Initial Slot:
Fault Injection Timing (if any):
Procedure:
Expected Device Behavior:
Independent Oracle / Evidence:
PASS / FAIL Criteria:
```

另請回答：如何確認斷電確實發生在你要測的更新階段，而不是僅依測試程式「已送出斷電指令」就算完成故障注入？

## (3) Release Gate｜功能符合性與根因調查如何分開？

**A.** 依目前資料，請選擇 `Go`、`Hold` 或 `No-Go`，說明依據；不要把「尚未找到唯一根因」與「尚未觀測到規格問題」視為同一件事。

**B.** 假設 RD 修正後，OTA Dashboard、Agent 與 CI 再次全部顯示 `SUCCESS`。請提出**最少兩項彼此獨立、可追溯至同一次開機的證據**，用來重新驗證 REQ-01，並說明各自的 PASS／FAIL 判準。

**C.** 如果 REQ-01 已通過修正後驗證，但 REQ-02、REQ-03 仍未執行，你會如何記錄這兩項需求的驗收狀態，以及如何處理 Release Gate？

---

**作答方式：** 請在本題的對話分支依序回答 (1)～(3)。本次不附答案或解析；收到作答後，再針對本題解析核心觀念、證據界線、常見誤區與排查策略。
