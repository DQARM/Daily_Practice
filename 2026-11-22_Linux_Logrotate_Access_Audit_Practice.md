# 今日技術練習｜2026/11/22

**主題：Linux / Security × logrotate × File Permission × Audit Integrity**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜Log Rotation 後服務仍正常，為什麼稽核資料卻可被一般帳號讀取？

你負責驗證隔離 LAB 中的 Linux 報表 API `reportapi`。服務以 `appsvc` 帳號執行，將每次成功處理的請求寫入 `/var/log/reportapi/events.log`。維運團隊新增了 `logrotate` 規則，準備發布到正式環境。

所有帳號、事件、時間、位址與檔案內容均為**原創虛構 LAB 資料**，不含真實個資。測試只在本機檢驗讀取權限及稽核紀錄，不連接正式環境。

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義 | 本題用途 |
|---|---|---|
| `logrotate` | Linux 日誌輪替工具 | 依設定將現有日誌改名、建立新日誌，並管理舊檔。 |
| DAC | Discretionary Access Control；自主存取控制 | Linux 透過檔案擁有者、群組與權限位元等資訊控制存取。 |
| POSIX ACL | Portable Operating System Interface Access Control List；存取控制清單 | 可在基本擁有者／群組／其他使用者權限以外，定義額外存取規則。 |
| `0640`／`0644` | 八進位檔案權限表示法 | 分別描述擁有者、群組及其他使用者的讀寫權限。 |
| `stat`／`getfacl` | 查詢檔案中繼資料／查詢檔案 ACL | 用來檢查檔案權限、擁有者及額外 ACL 項目。 |
| Inode | 索引節點 | Linux 檔案系統中識別檔案物件的中繼資料結構；路徑名稱可能變更。 |
| `create`／`postrotate` | logrotate 建立新檔／輪替後動作 | 本題設定檔內的兩個指令區段。 |
| `SIGHUP` | Hangup Signal；掛斷訊號 | 在本 LAB 中，服務收到此訊號後會重新開啟日誌檔。 |
| Audit Trail／Event ID | 稽核軌跡／事件識別碼 | 用來將成功請求與持久保存的記錄逐筆關聯。 |
| Test Oracle／Release Gate | 獨立測試判準／發布關卡 | 用核准需求判定結果，並決定版本能否發布。 |

### 一、已核准的驗收要求

本 LAB 已確認 `/var/log/reportapi` 目錄的權限為 `0755 root:root`，沒有額外 ACL。`visitor` 是一般本機帳號，不屬於 `appsvc` 或 `audit` 群組，且沒有 `sudo`、特權 Capability 或其他旁路讀取權限。

| ID | Requirement |
|---|---|
| REQ-01 | `visitor` 不得透過一般檔案存取方式讀取**現行或保留的任何一份**報表稽核日誌；授權的 `audit` 群組則必須能讀取。 |
| REQ-02 | 每筆回傳 HTTP `200` 的請求均須在回應後 **5 秒內**留下**恰好一筆**持久化稽核事件。事件需以獨立的 `run_id` 與 `event_id` 關聯，不得漏記或重複。 |
| REQ-03 | 每次輪替後，新 `events.log` 與保留的 `events.log.1` 均須符合 `0640 appsvc:audit`；服務重啟及第二次輪替後也須維持適當權限。 |
| REQ-04 | Release Gate 須結合獨立成功請求清單、完整事件集合、以受限帳號實際存取的結果、檔案權限／ACL 與輪替前後 Inode 證據；不得只依服務存活、logrotate 執行成功或 CI PASS 放行。 |

### 二、設定與測試方式

維運提供的 `logrotate` 規則如下；`reportapi` 在收到 `SIGHUP` 時會重新開啟其日誌檔。測試期間沒有其他服務寫入這個日誌目錄。

```text
/var/log/reportapi/events.log {
    rotate 4
    size 1M
    missingok
    create 0644 appsvc appsvc
    postrotate
        /bin/systemctl kill -s HUP reportapi.service
    endscript
}
```

QA 使用外部請求產生器，在同一個 `run_id=LIN-1122-01` 依序送出 `EVT-001`～`EVT-120` 共 120 筆請求。每筆收到 HTTP `200` 後才發送下一筆；在其中一段時間執行受控 logrotate。QA 於最後一次成功回應 **10 秒後**封存日誌快照。

### 三、觀測結果

| 觀測點 | Evidence |
|---|---|
| 外部請求紀錄 | 120 個不同 `event_id`，全部 HTTP `200`；完整回應時間已保存 |
| 輪替前檔案 | `events.log`：Inode `81201`，`0640 appsvc:audit` |
| 輪替前帳號測試 | `visitor` 讀取遭拒；`audit` 群組測試帳號可讀取 |
| logrotate 執行紀錄 | 指令結束狀態 `0`，且 `SIGHUP` 發送結果為成功 |
| 輪替後 `events.log.1` | Inode `81201`，`0640 appsvc:audit` |
| 輪替後新 `events.log` | Inode `81299`，`0644 appsvc:appsvc` |
| 輪替後帳號測試 | `visitor` 可讀取新 `events.log` 中的測試事件 `EVT-119` |
| 稽核事件獨立比對 | 兩份日誌合計 **119 個不同 `event_id`**，沒有重複；`EVT-087` 未找到 |
| 測試擷取品質 | 原始請求清單與兩份日誌內容均已保存；獨立解析器沒有回報檔案讀取錯誤 |
| CI Smoke Test | `SERVICE_ACTIVE=PASS`、`LOGROTATE_EXIT_0=PASS`、`HTTP_200=PASS`，合計 `3/3 PASS` |
| 第二次輪替／重啟 | 尚未執行；完整程序開檔描述符、服務內部日誌佇列狀態尚未取得 |

QA 補充查驗了新檔的權限：

```text
$ stat -c '%i %a %U:%G %n' /var/log/reportapi/events.log
81299 644 appsvc:appsvc /var/log/reportapi/events.log

$ getfacl -cp /var/log/reportapi/events.log
user::rw-
group::r--
other::r--
```

RD 認為：「`reportapi` 正常、`logrotate` exit code 是 0，而且 CI 3/3 PASS。少一筆可能只是記錄時間差；權限是 Linux 系統管理的事，可以先 Release。」

---

## (1) Evidence Analysis｜正常輪替與符合安全需求是同一件事嗎？

**A.** 請對 REQ-01～REQ-04 各標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，並說明哪些是**已觀測的規格結果**、哪些是**尚未證明的根因**、哪些是**尚未執行的測試**。

**B.** `logrotate exit 0`、`SIGHUP 成功`、Inode `81201`／`81299`、`getfacl`、`visitor` 實際讀取、以及 `CI 3/3 PASS` 各能證明什麼、不能證明什麼？請區分「檔案存在」、「資料可持續寫入」、「存取控制符合需求」及「稽核資料完整」。

**C.** 只有目前的觀測，能否斷定 `EVT-087` 缺失的**唯一根因**是 logrotate？若要釐清遺失發生的位置或時點，仍需要哪些可關聯證據？

## (2) Verification Design｜最多兩個高鑑別度流程

你可使用隔離 VM Snapshot、受控 logrotate、服務重啟、不同權限的測試帳號、獨立 HTTP 請求產生器以及檔案／程序／日誌監測；不得使用真實敏感資料、放寬 `0640` 或刪除驗收需求。

請設計**最多兩個**驗證流程，同時涵蓋日誌輪替前後的存取控制、稽核事件恰好一次保存，以及服務重啟／第二次輪替的持續性。請說明如何避免不同測試流程互相污染狀態。

每個流程請填寫：

```text
Verification Flow 1 / 2:
Initial State / Snapshot / run_id:
Controlled Operation (rotation / restart / permission test):
Independent Request and Audit Oracles:
File Mode / ACL / Inode / Process Evidence:
Timing and Correlation:
PASS / FAIL Criteria:
Evidence to Preserve:
```

## (3) Release Gate｜修正設定後如何避免假綠燈？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`；請區分「已觀測的違規」、「尚未確認的根因」與「缺少的回歸範圍」。

**B.** 假設 RD 修改設定後，下一次 `logrotate` 顯示成功、`systemctl is-active reportapi` 顯示 `active`，CI 仍是 `3/3 PASS`。你要求哪**兩組來源不同、可關聯到相同版本與 `run_id`** 的證據，才能重新評估 REQ-01～REQ-04？請寫出資料來源與 PASS／FAIL 判準。

**C.** 請設計一個安全、可還原的 **Negative Control（負向對照）**，證明新增測試確實能攔下「輪替後日誌可被非授權帳號讀取」或「成功請求缺少稽核事件」的版本，而不只是再次回報 CI PASS。寫出受控違規條件、預期偵測反應及復原方式。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪只提供題目及理解題意所需的名詞解釋，不附答案、提示性解答或解析；作答後再逐項批改證據判讀、測試設計與 Release Decision。
