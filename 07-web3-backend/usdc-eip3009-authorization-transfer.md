# USDC EIP-3009：`transferWithAuthorization` / `receiveWithAuthorization`

## 人話
CEX 後台好熟「用戶簽一張 off-chain 單 → 你哋扣帳／入帳」。USDC 嘅 EIP-3009 就係鏈上版：用戶用 EIP-712 簽一張**授權轉帳**，**唔使自己有 ETH 付 gas**；任何人（通常係交易所 hot wallet / relayer）可以代為上鏈，USDC 先由 `from` 轉去 `to`。
同舊式 `approve` + `transferFrom` 比：唔使兩步、唔使永久 allowance；同 EIP-2612 `permit` 比：permit 仍然係「授權 spender」，之後仲要 `transferFrom`；3009 係**一次簽名直接轉走金額**。

## 面試短答
- **核心**：EIP-712 typed data 簽 `TransferWithAuthorization`（或 receive 變體）；欄位大致：`from, to, value, validAfter, validBefore, nonce`；domain 綁 `name/version/chainId/verifyingContract=該鏈 USDC`。
- **`transferWithAuthorization`**：任何人都可以 relay（`msg.sender` 唔使係 `to`）→ 適合 gasless 入金／partner 結算；但有 frontrun 搶跑風險（惡意 relayer 搶先提交，錢仍然去正確 `to`，不過時序／配對可能亂）。
- **`receiveWithAuthorization`**：要求 `msg.sender == to` → 只有收款方自己可以上鏈「收」→ CEX 用自己地址當 `to` 時，只有交易所能 relay，**防別人搶先把授權用掉搞亂入帳時序**。
- **Replay**：每個 `from` 有獨立 `nonce`（bytes32，合約記「用過即廢」）；過期靠 `validBefore`；`validAfter` 可延遲生效。
- **CEX 場景**：用戶簽授權把 USDC 拉入交易所充值地址（gasless deposit pull）；OTC／partner settlement 對方簽、你方 hot wallet 付 gas。
- **入帳**：scanner **仍然以實際 `Transfer` event（或餘額變動）為準**；`AuthorizationUsed` 只係輔助關聯，唔可以只聽授權事件就入帳。
- **風險**：簽錯 `to`／`value`／錯 chain domain = 錢可被合法轉走；前端要清晰展示 domain + 欄位；熱錢包／relayer 付 gas、要有限額同監控。

## 常見追問（連答案）
1. **點解唔繼續用 approve + transferFrom？**
   答：approve 要用戶先有 gas 簽一筆 tx（或者另走 permit）；而且 allowance 留喺鏈上有被無限額授權盜用嘅產品風險。3009 一次簽名、一筆 relay tx 就完成轉帳，額度同收款方寫死喺簽名入面，用完即廢（nonce）。
2. **`transferWithAuthorization` 同 `receiveWithAuthorization` 點揀？**
   答：要「誰都可以幫你上鏈」（例如公開 relayer、第三方支付 gas）→ transfer；CEX 充值想確保**只有自己能消耗呢張授權**、避免外人搶跑導致 scanner 時序混亂 → receive（`to` = 交易所地址，由交易所 relay）。
3. **nonce / 過期點防重放？跨鏈呢？**
   答：合約對每個 `from` 記錄用過嘅 `nonce`（bytes32），同一 nonce 再用會 revert。`validBefore` 過咗就無效。EIP-712 domain 含 `chainId` + `verifyingContract`，簽畀 Ethereum 主網 USDC 嘅授權**唔可以**拎去另一條鏈／另一個 USDC 合約重放。
4. **用戶簽咗但未上鏈，CEX 可唔可以先入帳？**
   答：唔好。簽名只係授權意圖；鏈上未成功 `Transfer` 之前資產仲喺用戶地址，可能被用戶另途轉走或授權過期。流程：收簽名 → 校驗 domain/欄位/ecdsa → 狀態 `auth_pending` → relayer 上鏈 → 等確認 → 見到 `Transfer(from→我方)` 先 ledger credit。失敗／過期就拒單，唔扣用戶鏈下餘額（呢個係 pull 入金，本來就未入帳）。

## 小練習（題目 + 參考答案）
**題目**：產品要做「用戶無 ETH 都可充 USDC」。設計簽名欄位同後端狀態；另一個工程話「聽 `AuthorizationUsed` 就入帳快啲」。你點回應？會用 transfer 定 receive？

**參考答案**：
- 用 **`receiveWithAuthorization`**，`to` = 交易所該鏈 USDC 收款地址（或用戶專屬 deposit 合約），由交易所 relayer／hot wallet 做 `msg.sender`，避免外人搶跑。
- 簽名欄位：`from=用戶地址, to=收款地址, value, validAfter=0 或略延遲, validBefore=而家+N分鐘, nonce=伺服器發嘅唯一 bytes32`；domain 必須係該鏈官方 USDC。
- 後端：`requested → sig_verified → relaying → pending_on_chain → credited`；冪等鍵用 `(chainId, usdc, from, nonce)` 或業務 `depositAuthId`。
- **拒絕「只聽 AuthorizationUsed 入帳」**：授權用咗唔等於你司帳本應入帳金額／對象無誤（合約升級、錯誤監聽、夾心攻擊時序）。標準做法同普通 ERC-20 充值一樣——**以 `Transfer` 到我方地址（+ 確認數）為準**；`AuthorizationUsed` 只用來把 tx 同用戶授權單關聯、方便對帳同客服。
- Relayer：熱錢包付 gas、設每日 gas／金額上限、失敗重試要防雙重提交（同一 nonce 第二次會 revert，屬安全；但要標記終態）。
