# 今日技術練習｜2026/11/16

**主題：Linux / Security × systemd × File Capabilities × Independent Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜主服務顯示 NoNewPrivileges=yes，為什麼仍出現 UID 0 的處理程序？

你負責驗證隔離 LAB 中一台 Ubuntu 22.04 伺服器的報表服務 `reportd`。一般使用者透過 HTTPS 提交報表工作；主服務處理請求後，會透過受控的系統派送器啟動獨立的 `report-helper@<job-id>.service` 單元執行部分工作。**helper 是另一個 systemd service，不預設為主服務 fork 出來的子程序。**

所有帳號、檔案、時間與觀測都是本題的虛構測試資料，且僅使用測試用標記檔；不得在真實主機進行權限提升操作。

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義 | 本題中的用途 |
|---|---|---|
| systemd Unit | systemd 服務單元 | 定義一個服務如何啟動、以哪個身分執行及套用哪些限制。 |
| UID／EUID | User ID／Effective User ID；使用者識別碼／有效使用者識別碼 | Linux 用來表示程序身分的欄位；有效身分用於部分權限判斷。 |
| `NoNewPrivileges`／NNP | No New Privileges；禁止取得新權限的執行旗標 | Linux 程序的安全性屬性，可透過 `/proc/<pid>/status` 的 `NoNewPrivs` 觀察。 |
| `execve()` | Execute Program；執行程式的系統呼叫 | 用來載入新程式，與程序身分、權限集合的變化有關。 |
| `CAP_SETUID` | Capability to change user IDs；變更使用者 ID 的能力 | Linux 將部分特權操作拆分後的一種 capability。 |
| File Capabilities／`getcap` | 檔案能力／查詢檔案能力的指令 | 用於檢視執行檔上的 `security.capability` 屬性。 |
| `cap_setuid=ep` | Permitted／Effective；允許／有效旗標 | 本題用於描述檔案附帶的 capability 設定；不等於任何程序必然取得相同權限。 |
| `CapEff`／`CapBnd` | Effective／Bounding Capability Set；有效／上限能力集合 | `/proc/<pid>/status` 中的程序權限集合欄位。 |
| `nosuid` | No Set-User-ID；掛載選項 | 影響掛載區域內特定執行檔特權機制的處理。 |
| `systemctl show`／`systemctl cat` | 查詢生效屬性／檢視 Unit 定義 | 用於觀察服務的載入設定與宣告內容。 |
| Audit Trace／Test Oracle | 稽核追蹤／測試判準依據 | 用來關聯操作紀錄，並對照獨立核准的預期結果。 |

### 一、已核准驗收需求

本 LAB 僅有 `reportd.service`、工作派送器與 `report-helper@.service` 參與報表工作；工作使用測試身分 `reportbot`（UID `1105`）。下列需求適用於每次 `POST /reports` 觸發的**整條執行路徑**。

| ID | Requirement |
|---|---|
| REQ-01 | 合法工作應回傳 HTTP `200`，且外部下載的報表 SHA-256 必須符合核准的獨立 Golden Artifact。 |
| REQ-02 | 主服務及其觸發的 helper 工作均須以 `reportbot` 執行；整個工作期間不得出現 UID／EUID `0`，也不得在 `execve()` 後取得新的 `CAP_SETUID` 特權。 |
| REQ-03 | 主服務及 helper 的實際執行程序均須有 `NoNewPrivs=1`；重新啟動服務及主機後仍須維持，不能只檢查 Unit 檔文字。 |
| REQ-04 | Release Gate 必須具備相同 `job_id` 的 API 結果、獨立報表雜湊、程序身分／capability／NNP 證據與已載入 Unit 設定；只看 `systemctl` 或 CI PASS 不足以放行。 |

### 二、目前設定與觀測

以下資料均來自重新開機後的一次完整工作，`job_id=JOB-1640`。Golden Artifact 的摘要代號 `H-GOLD` 代表獨立保存的完整 SHA-256，表中的代號僅為簡寫。

```ini
# systemctl cat reportd.service（節錄）
[Service]
User=reportbot
NoNewPrivileges=yes
ExecStart=/usr/local/bin/reportd

# systemctl cat report-helper@.service（節錄）
[Service]
User=reportbot
ExecStart=/usr/local/libexec/report-helper --job=%i
# 節錄中未宣告 NoNewPrivileges
```

```text
$ systemctl show reportd.service -p User -p NoNewPrivileges
User=reportbot
NoNewPrivileges=yes

$ systemctl show report-helper@JOB-1640.service -p User -p NoNewPrivileges
User=reportbot
NoNewPrivileges=no

$ getcap /usr/local/libexec/report-helper
/usr/local/libexec/report-helper cap_setuid=ep

$ findmnt -T /usr/local/libexec/report-helper -o OPTIONS -n
rw,relatime
```

QA 於工作進行中保存以下彼此關聯的紀錄：

| Evidence | Observation |
|---|---|
| 外部 `POST /reports` | HTTP `200`，回應 `job_id=JOB-1640` |
| 外部下載報表 SHA-256 | `H-GOLD`，與獨立 Golden Artifact 相符 |
| `reportd` 主程序 `/proc/4100/status` | `Uid: 1105 1105 1105 1105`、`NoNewPrivs: 1`、`CapEff=0` |
| 工作派送器 | `JOB-1640` 對應 `report-helper@JOB-1640.service`，helper PID `4372` |
| helper `/proc/4372/status` 快照 | `Uid: 0 0 0 0`、`NoNewPrivs: 0`；`CapEff` 原始值已保存，但尚未解碼 |
| helper 稽核紀錄 | `job_id=JOB-1640`，觀察到 `setresuid(0,0,0)` 回傳 `0`；尚缺完整的 `execve()` 前後 capability trace |
| CI Smoke Test | `API_200=PASS`、`OUTPUT_HASH=PASS`、`MAIN_SERVICE_NNP=PASS` → `3/3 PASS` |
| Unit／程序完整性 | 尚未保存 helper 執行檔的部署摘要、完整 Unit 載入與重啟事件關聯 |

RD 認為：「`reportd` 已設定 `NoNewPrivileges=yes`，報表也正常，CI 3/3 PASS。helper 可能只是 `/proc` 擷取時序造成誤判；先 Release 就好。」

---

## (1) Evidence Analysis｜Unit 設定能代表整條執行路徑嗎？

**A.** 對 REQ-01～REQ-04 分別標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，逐項列出證據，並區分「已觀察到的狀態」、「尚缺的關聯證據」與「尚未確定的唯一根因」。

**B.** 比較 `reportd.service` 的 `NoNewPrivileges=yes`、helper Unit 的 `NoNewPrivileges=no`、`getcap cap_setuid=ep`、helper `Uid: 0 0 0 0`、`setresuid(...)=0`。各項能證明什麼、不能證明什麼？為何不能把**不同 systemd Unit** 的安全屬性視為同一個程序繼承鏈？

**C.** 單靠 `getcap` 與 `CapEff` 的片段資訊，能否重建 helper 是**何時、透過哪個機制**取得足以變更 UID 的權限？還需要哪些程序、執行檔、掛載或事件證據？

## (2) Verification Design｜最多兩個驗證流程

你可以在隔離 LAB 還原 VM Snapshot、使用虛構報表、建立測試專用 helper 複本，控制 Unit 安全設定及捕捉系統呼叫；不得使用真實機密、關閉主機整體安全防護或放寬 REQ-01～REQ-04。

請設計**最多兩個**具鑑別度的流程，驗證主服務與獨立 helper 的權限邊界，並區分下列假設：

| Hypothesis | Description |
|---|---|
| H1 | 不同 Unit 的生效限制或工作派送方式造成權限邊界差異。 |
| H2 | 執行檔 capability、掛載屬性或其他程序權限集合影響實際結果。 |
| H3 | PID 重用、擷取時間、事件關聯或量測方法造成觀察誤差。 |

每個流程填寫：

```text
Verification Flow 1 / 2:
Initial State / Controlled Variables:
Unit / Executable / Mount Configuration:
Procedure and Job Correlation:
Runtime Measurement (UID, EUID, NoNewPrivs, capabilities, execve):
Independent Oracle:
PASS / FAIL Criteria:
How Results Distinguish H1 / H2 / H3:
Rollback and Evidence:
```

## (3) Release Gate｜主服務的綠燈足以放行嗎？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`，根據 REQ-01～REQ-04 說明。請區分已觀測的安全性問題與仍需調查的細部因果機制。

**B.** 假設 RD 調整 Unit 後，`reportd` 與 helper 的 `systemctl show` 都顯示 `NoNewPrivileges=yes`，而 API 與報表雜湊仍 PASS。你還要求**哪兩組來源不同、且能對應同一次 `job_id` 與重啟後版本**的獨立證據，才會重新評估 Release？寫出來源與判準。

**C.** 設計一個可安全還原的 **negative control（負向對照）**，驗證回歸測試能攔下「主服務安全設定看似正確，但另一個 Unit 的 helper 取得禁止的權限」的版本。寫出已知違規條件、量測方式、預期測試反應與復原方法。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪僅提供題目與理解題目所需的名詞解釋，不附答案、提示性解答或解析；作答後再逐項批改證據判讀、驗證設計、常見誤區與排查思路。
