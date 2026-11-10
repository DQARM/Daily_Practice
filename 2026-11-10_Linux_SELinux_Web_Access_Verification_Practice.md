# 今日技術練習｜2026/11/10

**主題：Linux / Security × SELinux × Apache Access Control**  
**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題**

## 情境｜修好網站的 403，會不會同時讓敏感檔案曝光？

你負責驗證隔離 LAB 中一台 RHEL 9 伺服器 `app01` 的 Apache HTTP Server（`httpd`）。部署團隊將靜態網站移到 `/srv/portal`，目前外部使用者無法開啟首頁。網站目錄內另有一份**只含虛構測試資料**的內部備份檔，依需求不得透過 HTTP 讀取。

本題所有路徑、時間及結果均為虛構測試情境。`REQ` = *Requirement*（需求）；本題要驗證的是**修復後的實際存取控制**，不是只讓服務顯示正常。

### 一、驗收需求

| ID | Requirement |
|---|---|
| REQ-01 | 外部測試用戶端請求 `GET /index.html` 必須回傳 HTTP `200`，且回傳內容的 SHA-256 與獨立保存的已核准 Release Artifact 相符。 |
| REQ-02 | 外部用戶端不得經任何公開 HTTP 路徑取得 `/srv/portal/private/backup.json` 的內容；保護措施必須包含明確的 Web Server 存取限制，不能只依賴目前碰巧出現的 `403` 或 SELinux 阻擋。 |
| REQ-03 | SELinux 必須維持 `Enforcing`；僅對公開內容及必要目錄提供適當的 Web 讀取標籤，不得將整個敏感資料目錄視為公開 Web Content。重新開機及 `restorecon` 後仍須維持正確設定。 |
| REQ-04 | Release Gate 必須以**變更後**的外部請求、檔案／SELinux 狀態、Web Server 規則及可追溯日誌作為驗證證據；只看 `systemctl`、`getenforce` 或 CI PASS 不足以放行。 |

### 二、現有設定與觀測

Apache 設定摘要：

```apache
DocumentRoot "/srv/portal"
<Directory "/srv/portal">
    Require all granted
</Directory>
```

尚未發現任何針對 `/srv/portal/private` 的額外拒絕規則。檔案權限與安全標籤如下：

```text
$ getenforce
Enforcing

$ systemctl is-active httpd
active

# mode / owner / SELinux type
0755  root:root  default_t  /srv/portal
0644  root:root  default_t  /srv/portal/index.html
0755  root:root  default_t  /srv/portal/private
0644  root:root  default_t  /srv/portal/private/backup.json
```

QA 在另一台用戶端上取得的 HTTP 與稽核證據：

| 時間 | Evidence | Observation |
|---|---|---|
| 11:02:01 | `GET /index.html` | HTTP `403` |
| 11:02:02 | `GET /private/backup.json` | HTTP `403` |
| 11:02:01 | auditd AVC | `comm="httpd"`，讀取 `index.html`，`tcontext=...:default_t:s0`，`denied { read }` |
| 11:02:02 | auditd AVC | `comm="httpd"`，讀取 `backup.json`，`tcontext=...:default_t:s0`，`denied { read }` |
| 本次 CI | Smoke Test | `httpd=active`、`SELinux=Enforcing`、`private HTTP=403`，共 `3/3 PASS` |

RD 預計執行下列修復，但**尚未執行，也沒有修復後的 HTTP 測試結果**：

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/portal(/.*)?'
sudo restorecon -RFv /srv/portal
```

RD 認為：「首頁的 403 是 SELinux 標籤問題；把整棵目錄改成 Web Content 後，首頁應該能開。內部備份檔先前也是 403，CI 已經測到，所以不必再測。」

---

## (1) Evidence Analysis｜目前的 403 能證明什麼？

**A.** 請逐項說明 `httpd=active`、`SELinux=Enforcing`、兩筆 HTTP `403`、兩筆 AVC、`3/3 PASS` 各自能證明什麼、不能證明什麼。尤其要區分「目前拒絕存取」與「修復後仍符合安全需求」。

**B.** 請對 REQ-01～REQ-04 各標記 `PASS`、`FAIL` 或 `NOT VERIFIED`，並說明判定依據。請將**已觀測事實**、**尚未發生的變更**及**可能的安全風險**分開敘述。

**C.** 單憑目前的資料，能否斷定 `backup.json` 已受到符合 REQ-02 的完整保護？在判斷 RD 提議的整樹重新標籤風險時，哪些設定與證據最重要？

## (2) Verification Design｜只允許兩個驗證流程

你可建立 VM Snapshot、使用測試專用備份檔、修改 LAB 中的 Apache／SELinux 設定並重新開機；不得關閉 SELinux，也不得將測試資料替換為真實憑證。

請設計**最多兩個流程**，同時驗證：公開首頁恢復正常、內部備份檔仍不可讀取，以及重新開機／`restorecon` 後設定仍然有效。每個流程請填寫：

```text
Test Flow ID:
Initial State / Snapshot:
Change or Fault Injection:
Remote HTTP Requests and Expected Status:
Independent Evidence (content hash / policy / labels / logs):
Persistence Check:
PASS / FAIL Criteria:
```

另外說明：若 `/private/backup.json` 回應 `403`，你會如何驗證**拒絕存取的原因**，避免將另一個偶然的 SELinux 錯誤當成已具備明確的 Web Server 保護？

## (3) Release Gate｜綠燈是否足以代表沒有曝露？

**A.** 依目前證據選擇 `Go`、`Hold` 或 `No-Go`，說明對 REQ-01～REQ-04 的判斷；不得將「推測修好後會成功」寫成已驗證結果。

**B.** 假設 RD 完成修復後，外部 `GET /index.html` 回傳 `200`，`GET /private/backup.json` 回傳 `403`，CI 再次顯示 `3/3 PASS`。請提出**兩項最有鑑別度、彼此獨立的補充驗證證據**，並寫出來源、時間／版本關聯及 PASS／FAIL 判準。

**C.** 請設計一個 LAB 中可安全執行的 **negative control（負向對照）**，證明你的回歸測試能偵測「私人檔案被意外暴露」，而不只是每次都得到漂亮的 PASS 報告。

---

**作答方式：** 請在本題對話分支回答 (1)～(3)。首輪不附答案、提示性解答或解析；作答後再逐項批改證據判讀、測試設計、常見誤區與排查思路。
