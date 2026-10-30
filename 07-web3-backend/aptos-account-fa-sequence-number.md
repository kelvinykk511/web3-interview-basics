# Aptos：帳戶、Fungible Asset 同 sequence number（充值 / 出金）

## 人話
Aptos 係 Move 系（同 Sui 同源），但 **Aptos 係帳戶模型**，比 Sui 更似 ETH：
- 每個帳戶有 **sequence number**（≈ ETH 嘅 nonce），出金要順序用。
- Token 而家主要係 **Fungible Asset（FA）** 標準，每個帳戶有個 primary fungible store（自動開）；舊式 `Coin<T>` 要先「register」先收到。APT 已經遷移到 FA。
- **Finality 好快**：交易一被 commit 就 final，唔使等確認數。

## 面試短答
> Aptos 係帳戶 + sequence number 模型，出金同 ETH nonce 一樣要單調分配，每筆交易仲有 `expiration_timestamp_secs`，過期就永遠唔會上鏈，所以超時可以等過期再查、安全重建。
> 充值掃鏈按 ledger version 游標掃已 commit 交易，先睇 `success == true`，再由 events / balance change 按資產識別入帳：FA 用 metadata object 地址、舊 Coin 用完整 type tag `0x1::aptos_coin::AptosCoin` 呢類，白名單精確匹配。地址一律 normalize 成 64 hex 長格式先比較。BFT 共識，committed 就 final，唔使確認數。

## 重點細節
- **地址**：32 bytes，`0x` + 64 hex。特殊地址可以寫短格式（`0x1`），同一地址會有長短兩種寫法 → 入庫、白名單、去重前一定 normalize（補零、小寫）。
- **Auth key rotation**：Aptos 帳戶可以換鑰匙，換咗之後「地址」唔再等於由公鑰推導出嚟嘅值 → 唔好用「公鑰推地址」嚟驗證外部地址歸屬；自己 HD 生成嘅充值地址冇問題。
- **帳戶要存在**：收款地址未建立時，轉 APT 通常會自動建帳戶（`aptos_account::transfer`）；舊式 Coin 收款方要先 register CoinStore，否則失敗 → 出金用 `aptos_account` 嘅 transfer 函數 / FA primary store，減少失敗。
- **Gas**：費用 = `gas_used × gas_unit_price`，交易要設 `max_gas_amount`；先 simulate 估 gas 再加 buffer。失敗交易仍然扣 gas。
- **Sequence number**：同 ETH nonce 坑一樣——中間一個卡住，後面全部等；建議每個熱錢包帳戶一個分配器（DB 行鎖 / Redis 原子遞增），失敗要用同一 sequence number 補位或者等過期。
- **Expiration**：交易必帶過期時間。設短啲（例如 30–60 秒），超時唔知結果 → 等過期後用 hash 查；查唔到就確定作廢，再用同一 sequence number 重建。
- **掃鏈游標**：Aptos 用全局單調遞增嘅 **version**（每筆交易一個號），比 block height 更細；斷點續掃用 version。亦可以用官方 indexer GraphQL（`fungible_asset_activities` 之類），但要同 node 數據對帳，唔好單信一個來源。

## 常見追問（連答案）
**Q1：同 Sui 有咩分別？**
A：Sui 係 object 模型，餘額係一粒粒 coin object，並發靠分開 gas coin；Aptos 係帳戶 + sequence number，並發瓶頸似 ETH nonce，靠多個熱錢包帳戶分流。兩者都係 BFT 快速 finality、都要精確匹配資產 type。

**Q2：點樣防假幣充值？**
A：FA 用 metadata object 地址做身份，舊 Coin 用完整 type tag（`<address>::<module>::<Struct>`）；同名 symbol 任何人可以發。白名單只認上述 ID。另外要分清 Coin 版同 FA 版係同一資產（遷移後 event 可能係 FA event），入帳邏輯兩邊都要識，但映射到同一個內部幣種。

**Q3：超時之後點處理先唔會出兩次錢？**
A：同一 sequence number 鏈上最多成功一筆，所以只要重建時**用返同一個 sequence number**，就唔會雙花；再加上 expiration，等過期查唔到即可安全重建。千祈唔好超時就分配新 sequence number 重發。

**Q4：入帳睇乜欄位？**
A：交易 `success`（`vm_status` 係 Executed successfully）、event 類型（FA `Deposit` / Coin `DepositEvent`）、store 屬於邊個 owner、金額同 decimals（FA metadata 有 decimals）。冪等鍵用 `version + event index`。

## 小練習（題目 + 參考答案）
**題：** 熱錢包 A 有 sequence 100–104 五筆出金，第 101 筆 RPC 超時，你哋 worker 自動用 sequence 105 重發 101 嗰張單。可能出咩事？應該點做？

**參考答案：**
- 如果原本嘅 101 其實已送到 mempool，之後照上鏈 → 同一張提現單被出兩次（101 同 105 都成功）。
- 如果 101 真係冇送到，102–104 會因為缺 101 卡住，直到 101 被補位或過期。
- 正確做法：提現單綁定 sequence number；超時先用 hash 查，查唔到就**用返 101** 重送同一份已簽交易（或者等 expiration 過咗，再用 101 重建）。只有確認 101 已被其他交易佔用 / 作廢，先可以改派新號。
