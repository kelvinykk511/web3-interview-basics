# BTC 充值地址派發：xpub 派生 + Gap Limit

> 銜接：[HD Wallet BIP-32/44](../02-crypto-wallets/hd-wallet-bip32-44.md)、[BTC UTXO 充值／出金](./btc-utxo-deposit-withdrawal.md)。

## Part A：xpub 只讀派地址（CEX 充值）

### 人話
BTC 充值地址通常用 HD 錢包按路徑派：`m/84'/0'/0'/0/i`（Native SegWit `bc1q…`）或 `m/86'`（Taproot）等。冷端／HSM 留 account 私鑰；熱端掃鏈服務只拿 **account-level xpub（或 zpub）**，按 `address_index` 算出地址寫入「地址 → userId」表。
CEX 類比：財務保險庫保管總帳密鑰；客服系統只有「可以印收據編號」嘅只讀編碼器——印得到收款碼，簽唔到付款。

### 面試短答
我會用帳本級 xpub 喺熱端派生充值地址，DB 記錄 `(path, address, userId, created_at, last_used_at)`。入帳掃描認 `txid:vout` 做冪等（見 UTXO 篇）。**熱端唔放可簽名私鑰**；歸集／出金走另一條 HSM／MPC。Path 要穩定可重放，災難恢復可以從 xpub + DB index 重算地址集合。

### 常見追問（連答案）

**Q1：xpub 洩漏有幾嚴重？**
A：攻擊者可以推導**所有**已派同未來可派嘅收款地址 → 用戶隱私／對帳關係暴露（邊個地址屬同一 account），但**單靠 xpub 唔能偷幣**。若同時洩漏任何一條 **non-hardened 子私鑰** + xpub，經典 BIP-32 風險可能推及其他子鑰——所以 account 層要用 hardened，熱端只拿 account 公鑰樹。xpub 仍屬機密：洩漏要輪換 account（新 xpub、停用舊地址、遷移）。

**Q2：點解 BTC 面試常問 gap limit？**
A：因為好多錢包軟體／探索器按 BIP-44 慣例：連續 `gap limit`（常見 20）個地址都未用過，就**停止往前掃**。CEX 如果「預先派生好遠」或「用戶跳過好多空地址先充」，外部錢包睇唔到餘額；你哋**自建 indexer 唔可以依賴 gap limit**，要以 DB 地址表為準全量訂閱／掃描。

**Q3：一人一址 vs 地址重用？**
A：BTC 隱私同找零慣例傾向少重用。CEX 常見：每用戶至少一個固定充值地址（UX 簡單），或按次輪換。輪換要寫清：舊地址仍要永久監聽（用戶可能收藏舊碼）；DB 唔刪地址映射。重用同一地址唔影響入帳正確性，但要接受鏈上可關聯。

**Q4：change 鏈（`/1/i`）同充值鏈（`/0/i`）？**
A：BIP-44 嘅 `change=0` 外部收款、`change=1` 找零。CEX 用戶充值只用 external；熱錢包出金／歸集嘅找零地址走 change 鏈，**唔好同用戶充值地址池混用**，避免掃鏈誤入帳或隱私混亂。

## Part B：Gap Limit 同 CEX Scanner 實務

### 人話
「Gap limit = 20」係**輕客戶端為咗唔使掃全世界**嘅啟發式，唔係共識規則。你喺交易所做充值，真相係：**所有曾經派畀用戶嘅地址都要聽事件**，即使中間隔咗 1000 個空地址。
類比：銀行唔會因為「連續 20 個帳戶冇入過錢」就唔再認第 21 個帳戶嘅轉帳。

### 面試短答
自建 BTC scanner：以 DB 嘅已派發地址集合為訂閱集（或區塊濾鏡／地址索引），**唔用 gap limit 做停止條件**。預派生策略：維持「已派發但未使用」池低於某水位就再派一批；每派一批就註冊進 indexer。恢復備份時從 xpub 按 DB 最大 index 重放，並做一次「DB 地址 vs 鏈上有 UTXO」對帳。

### 常見追問（連答案）

**Q1：產品想「一次預先生成 10 萬個地址」OK 嗎？**
A：技術上 xpub 派得，但 indexer 記憶體／布隆濾鏡／DB 會變大；空地址多亦增加運維面。較佳：按需派發 + 小預熱池（例如每用戶首次進入充值頁先派）。若必須預先大批量，要確保 scanner 按表加載，唔好假設 gap。

**Q2：用戶問「點解我 Electrum 睇唔到交易所俾我嘅地址餘額？」**
A：可能對方用 xpub 導入時 gap limit 停早咗，或者 path／地址類型（legacy vs segwit）唔一致。CEX 客服應指引用 txid 喺 block explorer 查，唔依賴用戶本地錢包發現。對內：我哋入帳唔依賴用戶錢包。

**Q3：點樣同「確認數／RBF」一齊設計？**
A：地址發現（xpub/gap）係「睇唔睇到」；確認數係「信唔信得過」。流程：scanner 認到 `txid:vout` → seen → N confirmations → credited；0-conf／RBF 風險見 UTXO 篇。兩件事分開，唔好用「錢包已顯示」當最終入帳。

## 小練習（題目後即附參考答案）

**題 1：** 審計發現熱端充值服務持有 `m/84'/0'/0'` 嘅 **xpriv**，方便「順便歸集」。你點評價？要點改？

**參考答案：** 不可接受——熱端被攻＝該 account 下所有充值 UTXO 可被掃空。應改為：熱端只持 xpub 派地址 + 寫掃描游標；歸集／出金用獨立簽名服務（HSM／MPC），策略引擎校驗目的地、額度、人工審批。歷史 xpriv 視為已洩漏風險：輪換新 account、遷移資金、監控舊地址。

**題 2：** 客訴「我向地址 index=500 充值，但內部工具用 xpub 掃只到 index=20，話冇錢」。錢仲喺鏈上。點處理？

**參考答案：** 立刻用 DB 映射確認該地址屬邊個 user；用 explorer／節點查 UTXO／txid 補入帳（冪等鍵 `txid:vout`）。修復 scanner：改為按 DB 全量地址或至少掃到 `max(address_index)+buffer`，**移除 gap limit=20 嘅停止邏輯**。對帳任務：所有已派發地址做一次鏈上餘額對齊，防止仲有漏單。
