# 今日技術練習｜2026/11/04

**主題：Linux / Security × sudoers × Effective Authorization Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜`sudo -l` 顯示只能重啟服務，為什麼操作員仍能修改系統檔案？

你是 QA／Security Tester，負責驗證隔離 LAB 中一台 RHEL 9 應用伺服器 `app01` 的最小權限設定。帳號 `deploy` 屬於 `ops` 群組，所有帳號、檔案與觀測均為虛構 LAB 資料。

### 一、驗收要求

| ID | Requirement |
|---|---|
| REQ-01 | `deploy` 只能透過 sudo 執行 `systemctl restart report-api.service`，不得透過 sudo 修改 `/etc/report-api/production.env`。 |
| REQ-02 | 其他非授權 `systemctl` 操作（例如 `stop` 或 `restart sshd.service`）必須遭拒。 |
| REQ-03 | 權限驗收必須包含實際操作與獨立稽核證據，不得只依賴 sudoers 文字或 `sudo -l`。 |

### 二、RD 提供的設定

`/etc/sudoers.d/report-api`：

```sudoers
%ops ALL=(root) NOPASSWD: /usr/bin/systemctl restart report-api.service
```

RD 執行：

```bash
sudo visudo -cf /etc/sudoers.d/report-api
sudo -l -U deploy
```

輸出摘要：

```text
/etc/sudoers.d/report-api: parsed OK
User deploy may run the following commands on app01:
    (root) NOPASSWD: /usr/bin/systemctl restart report-api.service
```

### 三、QA 取得的額外證據

`report-api.service` 的片段：

```ini
[Service]
User=reportapi
ExecStart=/usr/local/bin/report-api
EnvironmentFile=/etc/report-api/production.env
ExecStartPre=/usr/local/libexec/report-api-preflight
```

檔案與目錄權限：

```text
-rw-r--r-- root root  /etc/report-api/production.env
-rwxr-xr-x root root  /usr/local/bin/report-api
-rwxr-xr-x root root  /usr/local/libexec/report-api-preflight
```

另有以下觀測：

| Evidence | Observation |
|---|---|
| `deploy` 直接執行 `sudo systemctl stop report-api.service` | `not allowed to execute` |
| `deploy` 直接寫入 `/etc/report-api/production.env` | `Permission denied` |
| `deploy` 執行允許的 `sudo systemctl restart report-api.service` | Exit code `0` |
| 重啟前後 `/etc/report-api/production.env` 的 SHA-256 | **不同** |
| Audit Log | 顯示 `deploy` 執行了允許的 restart 命令 |
| 服務日誌 | `report-api-preflight` 在重啟期間執行 |
| `/usr/local/libexec/report-api-preflight` 的實際內容及執行身分 | **尚未核對** |
| 變更前後檔案內容、寫入程序與時間關聯 | **尚未核對** |

RD 認為：「sudoers 只允許 restart，直接寫設定檔也失敗，權限測試應該 PASS。Hash 改變可能是正常部署流程。」

## (1) Evidence Analysis｜命令限制與實際權限效果

**A.** `visudo -cf`、`sudo -l -U deploy`、直接寫入遭拒、允許的 restart 成功，分別能證明什麼、不能證明什麼？

**B.** 依現有證據，REQ-01～REQ-03 各應標記 `PASS`、`FAIL` 或 `NOT VERIFIED`？請明確區分「已觀測到的事實」「尚待確認的違規行為」「尚未證明的根因」。

**C.** `production.env` 的 SHA-256 在 restart 前後不同，是否足以斷定 `deploy` 已透過 sudo 越權修改檔案？請說明還需要哪些關聯證據。

## (2) Test Design｜只能增加兩個驗證動作

你可在隔離 LAB 建立快照、使用測試專用設定檔、檢視 systemd unit／程序權限、查詢 auditd 與檔案異動記錄，但不可影響真實生產環境。

請設計**兩個資訊價值最高的驗證動作**，區分：

| Hypothesis | Description |
|---|---|
| H1 | 允許的 restart 間接觸發了可修改受保護檔案的程序或腳本。 |
| H2 | 其他獨立的部署／排程程序造成檔案變更，與 `deploy` 的 restart 無關。 |
| H3 | 量測、時間對齊或檔案識別方式有誤。 |

每個動作請填寫：

```text
Verification 1 / 2:
Preconditions / Controlled Variables:
Commands / Measurement Points:
Expected Evidence:
How Results Distinguish H1 / H2 / H3:
```

不得只回答「看 Log」或「檢查權限」；須指出要取得的欄位、程序身分或事件關聯。

## (3) Release Gate｜如何判斷最小權限是否成立？

**A.** 現階段選擇 `Go`、`Hold` 或 `No-Go`，依據 REQ-01～REQ-03 說明理由；請分開處理「授權規格符合性」與「唯一根因是否已查明」。

**B.** 假設 RD 修正後，`sudo -l` 仍顯示同一條允許命令，且 restart 成功。請提出**兩項相互獨立的驗證證據**，證明允許的操作沒有造成未授權檔案修改，同時 REQ-02 仍有效。各自列出 PASS／FAIL 判準。

**C.** 如果負向測試與檔案完整性測試都通過，但缺少同一測試窗口的稽核紀錄，你如何記錄 REQ-03 的狀態？

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。本次不附答案或解析；收到作答後，再逐項評估證據判讀、驗證設計、常見誤區與排查思路。
