# EIP-712 Typed Data（結構化簽名）

## 人話

CEX 後端好熟：**HMAC／API key 簽請求**——簽嘅係明確欄位，唔係一串睇唔明嘅 bytes。  
鏈上早期好多簽名係 `eth_sign`／`personal_sign` 簽「任意字節」或純文字，用戶錢包只見到 hex，好易被騙簽。

**EIP-712** 規定用 **結構化 typed data** 簽名：有 `domain`（邊條鏈、邊個合約、版本）同 `types`＋`message` 欄位。錢包可以顯示「你係授權邊個 spender、幾多、deadline」，同 Permit／登入／訂單簿 off-chain 訂單同一套路。

類比：由「簽一串 base64 blob」升級成「簽一張有欄位名嘅業務表單」；重放防護靠 `chainId`、`verifyingContract`、同業務 `nonce`。

## 面試短答

EIP-712 係以太坊 **結構化數據簽名**標準：先 hash `domainSeparator` 同 `structHash`，再對 `\x19\x01 ‖ domainSeparator ‖ structHash` 做 ECDSA。  
Domain 通常含 `name、version、chainId、verifyingContract`，用來綁定鏈同合約，防跨鏈／跨合約重放。  
應用：EIP-2612 Permit、EIP-712 登入（SIWE 變體）、DEX off-chain order、多簽／治理投票。  
面試想聽：你懂點解唔好用「裸 personal_sign 任意訊息」做授權，同埋 domain 綁定同業務 nonce 嘅防重放角色。

## 常見追問（連答案）

**Q: EIP-712 同 `personal_sign` 差在哪？**  
A: `personal_sign` 多係簽一段文字／bytes，錢包展示弱、易被釣魚成「簽咗等於授權」。712 有類型同欄位，錢包可結構化展示；合約端用固定 encoding 還原 hash 再 `ecrecover`。授權類場景優先 712。

**Q: Domain 入面 `chainId`／`verifyingContract` 漏咗會點？**  
A: 簽名可能喺另一條鏈或另一個合約被重放（若業務邏輯唔再校）。正確設計：domain 綁死目標鏈同驗證合約；合約內重建同一個 domainSeparator 再驗簽。

**Q: 同帳戶 tx nonce、Permit `nonces(owner)` 有咩關係？**  
A: 三套常見「序號」：① 帳戶發 tx 嘅 nonce（mempool／鏈上排序）；② Permit／訂單等業務 nonce（防同一張 712 簽名重放）；③ 有時仲有 session／device nonce。面試分開講，唔好混成一個 nonce。

**Q: CEX 後端何時會碰到 712？**  
A: 對接支援 Permit 嘅 token、做「錢包簽名登入／綁定地址」、讀取用戶簽署嘅 off-chain 意圖再代交、或者內部風控要校用戶簽名域有冇綁對 `chainId`／合約。熱錢包自己 `signTransaction` 出金唔等於 712；712 多係 **用戶／對手方簽業務意圖**。

## 小練習

**題：** 產品要「用錢包簽名綁定 CEX 帳戶」。對手丟嚟一條要你 `personal_sign` 嘅隨機 hex，話係登入挑戰。你點拒？正確 EIP-712 方案要包含邊啲欄位？

**參考答案：**  
拒簽無語義／純 hex 挑戰（釣魚高發）。改用 EIP-712（或 SIWE 類）：domain 含 `name=你哋產品、chainId、version`；message 含 `cexUserId`（或地址綁定意圖）、`nonce`（一次性）、`issuedAt`／`expiration`。後端只認自己 domain、校 nonce 只用一次、校過期時間；絕不要求用戶簽「可能被解讀成授權轉幣」嘅模糊訊息。
