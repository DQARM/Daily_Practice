# 今日技術練習｜2026/10/29

**主題：Linux / Security × OpenSSH Hardening × Effective Configuration Verification**  
**難度：★★★★☆｜預估 10–15 分鐘｜僅出題，不附答案**

## Q1｜`sshd -T` 顯示已停用密碼登入，為什麼測試帳號仍能用密碼登入？

你負責驗證一台 Rocky Linux 9 的 SSH Jump Server（`jump01.lab`）。這是完全授權的隔離 LAB，測試帳號與 IP 均為虛構。

### 一、Security Requirements

| ID     | 驗收要求                                                                          |
|--------|-----------------------------------------------------------------------------------|
| REQ-01 | 只有 `ssh_admins` 群組成員可以透過 SSH 登入。                                     |
| REQ-02 | 所有來源 IP 都不得使用 **password-only** SSH 登入；允許已核准的 Public Key 登入。 |
| REQ-03 | `root` 不得直接透過 SSH 登入。                                                    |
| REQ-04 | 設定重新載入及主機重開機後，以上限制仍須有效。                                    |

### 二、RD 提供的設定與檢查

RD 出示 `/etc/ssh/sshd_config` 的相關內容：

``` sshconfig
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowGroups ssh_admins
Include /etc/ssh/sshd_config.d/*.conf
```

CI 執行以下檢查：

``` bash
sudo /usr/sbin/sshd -t
sudo /usr/sbin/sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|allowgroups) '
```

輸出：

``` text
# sshd -t: exit code 0
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
allowgroups ssh_admins
```

RD 表示：**「語法檢查通過，而且有效設定已停用密碼登入。」**

### 三、QA 的額外證據

**Evidence A｜依連線條件查詢**

``` bash
sudo /usr/sbin/sshd -T \
  -C user=ops01,host=jump01.lab,addr=10.50.8.23 \
  | grep -E '^(passwordauthentication|permitrootlogin|allowgroups) '
```

``` text
passwordauthentication yes
permitrootlogin no
allowgroups ssh_admins
```

**Evidence B｜真實連線測試**

測試帳號 `ops01` 已確認屬於 `ssh_admins`，測試端沒有提供 Public Key，且 Client 明確限制只嘗試 Password Authentication。

| 測試來源     | 登入帳號 | 驗證方式      | 觀測結果              |
|--------------|----------|---------------|-----------------------|
| `10.50.8.23` | `ops01`  | Password only | SSH Session 建立成功  |
| `10.50.9.23` | `ops01`  | Password only | Authentication failed |

**Evidence C｜Server Log（與成功連線的時間、來源 IP 對得上）**

``` text
sshd[4182]: Accepted password for ops01 from 10.50.8.23 port 53120 ssh2
```

**Evidence D｜尚未確認的事項**

``` text
/etc/ssh/sshd_config.d/ 目錄存在，但尚未檢視所有檔案內容。
尚未核對實際監聽 Port 22 的 sshd Process 啟動參數。
尚未執行 root / 非 ssh_admins 帳號的負向測試。
尚未執行重開機後驗證。
```

------------------------------------------------------------------------

## (1) Evidence Analysis｜設定、有效設定與實際行為

請回答：

**A.** `sshd -t`、不帶 `-C` 的 `sshd -T`，以及帶有 `-C user=...,host=...,addr=...` 的 `sshd -T`，三者各能證明什麼？各自不能證明什麼？

**B.** 目前是否已有足夠證據判定 **REQ-02** 不符合？你會如何界定「已確認的違規行為」與「尚未確認的設定根因」？

**C.** 目前的證據能否直接宣稱 **REQ-01、REQ-03、REQ-04** 均符合？請指出證據缺口。

## (2) Troubleshooting｜兩個最有資訊價值的檢查

你需要區分以下假設：

| 假設 | Description                                                             |
|------|-------------------------------------------------------------------------|
| H1   | 某個額外載入的 SSH 設定，對特定來源條件改變了認證行為。                 |
| H2   | 實際提供 SSH 服務的 Process，其設定來源或啟動參數與 QA 檢查的預期不同。 |
| H3   | QA 的 Client / Log 關聯或驗證方式判讀有誤。                             |

**只能優先執行兩個驗證動作。** 每個動作請說明執行位置、具體指令或量測方式、預期觀測欄位，以及不同結果如何支持或排除 H1～H3。

``` text
Verification 1
執行位置：
指令／證據來源：
預期觀測：
如何區分 H1 / H2 / H3：

Verification 2
執行位置：
指令／證據來源：
預期觀測：
如何區分 H1 / H2 / H3：
```

**限制：** 不能只回答「看設定檔」或「看 Log」；必須具體指出要查什麼，以及觀測結果如何改變判斷。

## (3) Release Gate｜修正後如何避免只測到一條路徑？

RD 提議移除疑似造成例外的設定，重新載入 SSH Service，並再次執行 `sshd -t` 與不帶 `-C` 的 `sshd -T`。

**A.** 對修正前的版本，你會選擇哪個 Release Decision？請以需求與已取得的證據說明。

- **Go**：CI 的設定檢查通過，且大多數來源不能用密碼登入。
- **Hold**：需先查明唯一根因，才能判斷是否符合驗收要求。
- **No-Go**：目前已有足以支持不符合明確驗收要求的行為證據。

**B.** 請設計一份**最小但有鑑別度的修正後驗證矩陣**，至少考慮：來源 IP、帳號所屬群組、登入身分（含 root）、驗證方式。請列出各類測試的 Expected Result 與應保留的獨立證據。

**C.** 如果修正後的測試全數 PASS，但尚未做 Reboot Test，你會如何處理 **REQ-04** 的驗收狀態與 Release Decision？

------------------------------------------------------------------------

**作答原則：** Evidence → Hypothesis → Verification → Release Decision。請區分靜態設定、依連線條件計算的有效設定，以及實際連線結果。本次先不提供答案或解析；請在本題的對話分支中作答。
