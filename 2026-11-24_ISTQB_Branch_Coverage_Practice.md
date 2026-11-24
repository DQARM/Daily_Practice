# 今日技術練習｜2026/11/24

**主題：ISTQB-CTFL × Statement Coverage × Branch Coverage × Boundary Value Analysis**

**難度：★★★★☆｜預估 10–15 分鐘｜首輪只出題｜滿分 100 分**

## 情境｜程式分支覆蓋率 100%，偏遠地區的運費卻算錯了

你負責驗證電商系統的運費計算模組。RD 表示所有程式分支均已執行，而且既有單元測試全部 PASS，因此建議直接發布。本題使用原創虛構的需求、程式與測試資料，不涉及真實訂單或金流。

### 名詞與縮寫速查

| 名詞 | 英文全名／中文意義 | 本題用途 |
|---|---|---|
| ISTQB-CTFL | International Software Testing Qualifications Board — Certified Tester Foundation Level；國際軟體測試資格認證基礎級 | 本題的學習範圍。 |
| Statement Coverage | 敘述覆蓋率 | 衡量測試執行過的可執行敘述比例。 |
| Branch Coverage | 分支覆蓋率 | 衡量控制流程中已執行的分支比例，包含條件式與非條件式分支。 |
| Execution Path | 執行路徑 | 一次程式執行依序經過的控制流程。 |
| EP | Equivalence Partitioning；等價劃分 | 將預期具有相同處理行為的輸入劃分成群組。 |
| BVA | Boundary Value Analysis；邊界值分析 | 針對有序等價區間的邊界選擇測試值。 |
| Test Oracle | 測試判準依據 | 用來決定預期結果及判定實際結果是否正確。 |
| Confirmation Testing | 確認測試 | 用來確認已回報的缺陷是否被修正。 |
| Regression Testing | 回歸測試 | 用來檢查變更是否對其他行為產生不良影響。 |
| CI／Release Gate | Continuous Integration；持續整合／發布關卡 | 分別用來執行自動檢查及評估發布條件。 |

### 一、已核准需求

所有金額均為新台幣整數；`subtotal` 是商品小計，不含運費。有效輸入範圍為 0～5000 元，`remote` 為布林值。無效輸入由上游模組處理，不在本題範圍內。

| ID | Requirement |
|---|---|
| REQ-01 | 商品小計未滿 1000 元，基本運費為 80 元；滿 1000 元（含）時，基本運費為 0 元。 |
| REQ-02 | 偏遠地區（`remote=True`）須在基本運費之外加收 120 元；一般地區不加收。 |
| REQ-03 | 發布前必須驗證四種業務條件組合，以及 1000 元門檻的三值邊界（999、1000、1001），兩種地區皆須涵蓋。所有驗收結果須符合 REQ-01、REQ-02。 |

### 二、目前版本的完整受測函式

```python
def shipping_fee(subtotal, remote):
    if subtotal >= 1000:     # D1
        base = 0            # S1
    else:
        base = 80           # S2

    if remote:               # D2
        fee = 120           # S3
    else:
        fee = base          # S4

    return fee              # S5
```

`D1`、`D2` 是兩個判斷節點；`S1`～`S5` 是本題標記的五個賦值或回傳敘述。計算分支時，請先列出四個條件分支結果，再檢查其他非條件式控制流程是否也已走過。函式沒有迴圈、例外或其他隱藏分支。

### 三、測試紀錄

| Test ID | subtotal | remote | 預期運費（依需求） | 實際運費 | CI 結果 |
|---|---:|---|---:|---:|---|
| T1 | 500 | False | 80 | 80 | PASS |
| T2 | 1500 | True | 120 | 120 | PASS |

CI 報告顯示 `statement coverage=100%`、`branch coverage=100%`、`2/2 PASS`。

QA 另外執行一筆未納入 CI 的測試：`T3: subtotal=999, remote=True`，實際回傳 120 元。QA 尚未填入 T3 的預期結果與判定，也尚未執行完整的邊界測試。

RD 表示：「兩個 if 的 True、False 都走過了，沒有任何程式分支沒測到，所以 REQ-01～REQ-03 應該都已通過。」

---

## 第 1 題｜白箱測試與覆蓋率（30 分）

A. 分別列出 T1、T2 執行的判斷結果與 S1～S5 敘述。計算合併後的標記敘述覆蓋率及條件分支覆蓋率，並判斷 CI 宣稱的完整分支覆蓋率是否成立。

B. 列出這個函式所有可行的 D1／D2 結果組合，指出 T1、T2 實際涵蓋哪些執行路徑。分支覆蓋與執行路徑覆蓋是否等價？請以本題說明。

## 第 2 題｜黑箱測試設計（45 分）

A. 不依賴受測程式作為預期結果來源，根據 REQ-01、REQ-02 建立完整的四規則決策表，列出各組合的正確運費。

B. 保留 T1、T2、T3，新增最少數量的測試案例，使合併後的測試集符合 REQ-03 所指定的組合與邊界覆蓋要求。請填寫下表，並補上 T3 的預期結果與 PASS／FAIL 判定。

| Test ID | subtotal | remote | Expected Fee | 對應需求及覆蓋項目 |
|---|---:|---|---:|---|
| 自行填寫 | | | | |

## 第 3 題｜缺陷管理與發布判斷（25 分）

A. 根據目前已取得的測試證據，選擇 Go、Hold 或 No-Go。說明 REQ-01～REQ-03 的驗證狀態，並區分未測試的情境與已觀測到的不符合項目。

B. 假設 RD 修正運費函式，請從你的測試集中指定確認測試與回歸測試的案例，說明兩者的目的、預期結果及重新放行的條件。

---

**作答方式：** 請在本題對話分支回答第 1～3 題。作答後再逐項批改、給分，並解析核心觀念及常見誤區。

**公開考綱參考：** [ISTQB® Certified Tester Foundation Level Syllabus v4.0.1（2024-09-15）](https://istqb.org/wp-content/uploads/2024/11/ISTQB_CTFL_Syllabus_v4.0.1.pdf)。本題為獨立原創練習，非官方試題。
