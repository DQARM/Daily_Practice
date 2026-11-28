# 今日技術練習｜2026/11/28

**主題：Linux / Security × Linux Audit × 稽核事件完整性**  
**難度：★★★★☆｜預估 10–15 分鐘｜滿分 100 分｜首輪只出題**

## 情境｜auditd 正常運作，為什麼安全稽核仍漏掉設定變更？

你負責驗證 Ubuntu 22.04 設備管理伺服器的 Linux Audit 功能。維運人員透過核准的管理工具更新測試設定檔；安全需求要求每一次成功更新都能追溯到原始登入者。RD 認為 auditd 服務正常、稽核規則已載入，而且測試結束時佇列為空，就代表稽核功能合格。

本題為原創隔離 LAB，帳號、檔案、事件及驗收門檻皆為虛構，不涉及真實機密或正式主機。

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義與用途 |
|---|---|
| auditd | Linux Audit Daemon；稽核常駐服務，負責接收核心稽核紀錄並保存。 |
| `auditctl -s`／`auditctl -l` | 分別查詢核心稽核子系統狀態與目前載入的規則。 |
| AUID／EUID | Audit User ID／Effective User ID；稽核登入者識別碼／有效使用者識別碼，用來描述事件中的不同身分。 |
| SYSCALL／PATH | 系統呼叫／路徑紀錄，描述操作及相關檔案物件。 |
| Audit Event Serial | 稽核事件序號，同一事件的多筆紀錄可透過事件識別資訊關聯。 |
| `key`／`success` | 稽核規則標籤／系統呼叫成功與否，用來分類及檢視事件。 |
| `lost`／`backlog` | 核心稽核遺失紀錄累計數／當前等待 auditd 讀取的紀錄數，用於觀察稽核子系統狀態。 |
| Test Oracle | 測試判準依據，用來判斷觀測結果是否符合核准需求。 |

## 一、驗收需求

測試帳號 `operator` 的 UID 為 1102；管理工具透過核准的 sudo 政策以 EUID 0 寫入 `/opt/lab-policy/settings.conf`。每次操作只執行一次受測的寫入系統呼叫，且保留唯一操作識別碼 `op_id`。測試工具的獨立紀錄已包含 PID、時間與操作結果，可用來對照稽核事件；`op_id` 並非 Linux Audit 的原生欄位。

| ID | Requirement |
|---|---|
| REQ-01 | 每次成功的設定更新都須有一筆可關聯的完整稽核事件，包含成功結果、目標檔案、原始登入者及有效執行身分。 |
| REQ-02 | 在每分鐘 20 次及 600 次更新的兩種核准負載下，測試期間核心稽核 `lost` 增量必須為 0。 |
| REQ-03 | 主機重新開機後，無須人工補載規則，仍須滿足 REQ-01 與 REQ-02。 |
| REQ-04 | 發布前須保存同一測試執行的獨立操作紀錄、原始稽核事件、核心狀態前後快照、載入規則與開機識別資訊。 |

## 二、測試觀測

本次只在同一次開機中執行低負載與高負載測試，每個窗口 60 秒，結束後等待 10 秒再統計。兩個窗口之間沒有重設計數器，測試帳號、目標檔案與管理工具版本均相同。

```text
$ systemctl is-active auditd
active

$ auditctl -l
-w /opt/lab-policy/settings.conf -p wa -k lab_policy

$ auditctl -s       # 高負載開始前，節錄
enabled 1
pid 812
lost 3
backlog 0

$ auditctl -s       # 高負載結束並收尾後，節錄
enabled 1
pid 812
lost 11
backlog 0
```

`-w` 指定監看的路徑，`-p wa` 選擇寫入及屬性變更類型，`-k` 設定規則標籤。以上均為本次 LAB 的既有配置觀測，不是要求你修改主機的指令。

| 觀測項目 | 低負載 L | 高負載 H |
|---|---:|---:|
| 獨立操作工具確認成功更新 | 20 | 600 |
| 可對應成功更新的完整稽核事件 | 20 | 594 |
| `lost` 開始值 → 結束值 | 3 → 3 | 3 → 11 |
| 收尾時 `backlog` | 0 | 0 |
| auditd 服務狀態（收尾時） | active | active |

高負載窗口的原始稽核紀錄節錄如下（省略不相關欄位，時間戳為虛構）：

```text
type=SYSCALL msg=audit(1795827600.100:2201): success=yes pid=5101 auid=1102 euid=0 key="lab_policy"
type=PATH    msg=audit(1795827600.100:2201): name="/opt/lab-policy/settings.conf"
```

獨立操作紀錄顯示 PID 5101、同一時間的成功更新對應 `op_id=H001`。其餘六筆缺少完整稽核事件的成功更新已有獨立操作紀錄，但尚未逐筆確認是核心、使用者空間保存或分析器哪一段造成缺失。持久規則檔與重新開機測試尚未取得。CI 僅檢查服務 active、規則存在及收尾 backlog 為 0，結果 3/3 PASS。

## 第 1 題｜身分與事件判讀（30 分）

**A.** 上述兩行稽核紀錄代表幾個事件？`auid=1102` 與 `euid=0` 分別描述什麼身分？能否僅憑 EUID 判定是哪個人發起操作？

**B.** 分別計算 L、H 的完整稽核事件覆蓋率與 `lost` 增量。高負載新增的遺失紀錄數能否直接等同於缺少的成功更新事件數？收尾 `backlog=0` 又能證明什麼？

## 第 2 題｜驗證設計與故障定位（45 分）

**A.** 對 REQ-01～REQ-04 各標記 PASS、FAIL 或 NOT VERIFIED，並指出證據及適用範圍。哪些是已觀測的違規，哪些仍屬未確認的因果關係？

**B.** 設計最多兩個可重複的測試流程，涵蓋兩種負載及重開機後的規則持續性，並說明如何區分核心稽核遺失、日誌保存問題與分析器關聯錯誤。可使用獨立操作工具、唯讀稽核查詢、時間戳與 VM 快照；不得關閉稽核功能、降低驗收負載或刪除既有日誌。

```text
Test Flow ID:
Initial State / Boot ID / Loaded Rules:
Workload / Observation Window:
Independent Oracle / Event Correlation:
Before and After Counters:
PASS / FAIL Criteria:
Evidence to Preserve:
```

## 第 3 題｜發布關卡（25 分）

**A.** 現階段選擇 Go、Hold 或 No-Go，並說明為何目前證據足以或不足以支持你的決定，不必先假定唯一故障根因。

**B.** 假設 RD 修正後 CI 仍是 3/3 PASS，你會要求哪些獨立回歸證據？另設計一個只使用合成紀錄副本的負向對照（Negative Control），驗證分析器能發現缺少完整事件或登入者身分的情況，且不修改原始稽核資料。

---

**作答方式：** 請在本題對話分支回答第 1～3 題。作答後再逐項批改、給分，並解析核心觀念、常見誤區與排查思路。

**公開文件參考：** [Ubuntu 22.04 LTS (Jammy) — auditctl(8)，auditd 3.0.7 系列手冊](https://manpages.ubuntu.com/manpages/jammy/man8/auditctl.8.html)。本題情境、數據及驗收要求均為獨立原創。
