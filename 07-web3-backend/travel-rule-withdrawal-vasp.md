# Travel Rule 出金：VASP 訊息 + 鏈上廣播閘門

## 人話
CEX 出金狀態機你已經識：凍款 → 風控 → 簽名 → 廣播。Travel Rule（旅行規則）係合規加嘅一閘：**達到門檻嘅轉帳，廣播上鏈之前**，要同對手方 VASP（另一間交易所／託管商）交換匯款人／收款人資訊。呢啲 PII **唔會、亦唔應該**寫上鏈 memo；鏈上仍然只係普通轉帳，合規資料走 **VASP-to-VASP 鏈下通道**（例如相關業界協議／合規網絡）。

## 面試短答
- **門檻**：FATF 建議框架下各地門檻**唔同**（有啲司法區有明確金額線、有啲更嚴或覆蓋幾乎所有轉帳）——面試講「睇牌照所在地／用戶所在地規則，系統要可配置，唔好寫死一個全球 $」。
- **資料**：高層類似 IVMS-101——originator（匯款人姓名／帳戶識別等）+ beneficiary（收款人）+ 金額／資產／VASP 識別；**最小化必要欄位**，按對手方可接收格式組包。
- **地址類型分流**：
  - 對手係 **hosted VASP 地址**（地址簿／belong-to 查詢知係邊間所）→ 走 Travel Rule 訊息，等對方 accept／ACK 先可進簽名廣播。
  - 對手係 **unhosted（自託管）錢包** → 唔係 VASP 對 VASP；改用所有權證明／用戶聲明等本地合規流程（見地址所有權驗證筆記），再按本地法決定能不能出。
- **狀態機加閘**：`requested → risk_approved → frozen → travel_rule_pending → approved → signing → broadcast → …`；需要 Travel Rule 時 **未成功前絕唔 broadcast**。失敗：`travel_rule_rejected`／`travel_rule_timeout` → 解凍或退回用戶，唔好「先打出去再補訊息」。
- **其他控管**：首次受益人地址冷靜期、地址簿白名單、大額升級人工；對手 VASP 不可達 → **hold 出金**，唔可以「把 PII 塞進鏈上 memo 當合規」。

## 常見追問（連答案）
1. **點解 PII 唔寫上鏈？**
   答：鏈上永久公開、不可撤回；Travel Rule 要嘅係受監管 VASP 之間可控、可審計嘅訊息交換同留存，唔係公開廣播身分。Memo／OP_RETURN 寫姓名地址係合規同安全雙重事故。
2. **Travel Rule pending 超時點算？可唔可以先廣播再補？**
   答：唔可以先廣播。超時 → 保持 `travel_rule_pending` 或轉 `held`，通知用戶／合規；對手恢復後重試同一 `withdrawId`（冪等）。已經廣播就無法「收回」鏈上轉帳，合規缺口補唔返。
3. **點知收款地址係咪另一間 VASP？**
   答：業界有地址歸屬查詢／VASP 目錄（具體供應商視公司接入）；查到就開 Travel Rule 通道；查唔到當自託管路徑處理。假陽性／假陰性要有人工覆核同更新機制，唔好只信一次快取永久。
4. **同現有 withdrawal SM／KYT 點疊？**
   答：KYT 係鏈上資金風險（污點地址）；Travel Rule 係身分／對手資訊交換。典型順序：業務校驗 → 凍款 → KYT／風控 →（若需）Travel Rule → 簽名廣播。任一閘拒絕都唔進入 signing；狀態同原因碼要可對帳、可審計。

## 小練習（題目 + 參考答案）
**題目**：用戶出金到「另一間大型交易所」地址，金額高過你司配置嘅 Travel Rule 門檻。合規 API 暫時 503。另一個同事建議：memo 寫上用戶姓名同收款人姓名先出金，稍後再補 VASP 訊息。你會點設計狀態同回覆？

**參考答案**：
- **拒絕 memo 方案**：PII 上鏈不可逆，亦唔被視為有效 Travel Rule 履行。
- 狀態停喺 `travel_rule_pending`（或 `held_compliance`），**唔進 signing／broadcast**；凍款繼續（或按產品政策限時解凍並關單——要產品／法務定，但重點係錢未上鏈）。
- 重試策略：指數退避打 VASP／合規網絡；同一 `withdrawId` 冪等，唔開第二筆凍款。
- 用戶側：顯示「合規核實中／稍後自動重試」，提供取消出金（取消 → 解凍）。
- 告警：合規通道 SLA 告警畀運維＋合規 on-call；恢復後自動 resume 同一單 → `approved` → 先簽名再廣播。
- 記錄：保留嘗試次數、對手 VASP id、錯誤碼、無 PII 嘅關聯 id，方便審計（PII 本体只存在合規系統，出金服務存 reference）。
