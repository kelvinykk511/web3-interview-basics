# EIP-1271：智能合約錢包簽名（CEX 綁定／授權）

## 人話（從你熟的 CEX）

CEX「綁定提現地址／登入簽名／離線授權」成日用 `personal_sign` 或 EIP-712，後端用 `ecrecover` 還原 EOA。但用戶好多時用 **Gnosis Safe／合約錢包**：鏈上冇單條私鑰對應「嗰個地址」，`ecrecover` 會對唔上。  
**EIP-1271** 就係合約自己提供 `isValidSignature(hash, signature) → magic value`，讓驗證方問：「呢個合約認唔認呢個簽？」

## 面試短答

- EOA：`ecrecover(hash, sig) == address`。
- 合約錢包：呼叫 `IERC1271(wallet).isValidSignature(hash, sig)`，回傳 `0x1626ba7e` 先算有效（實作可再包一層 Safe 的 threshold／owners）。
- CEX 後端綁定流程：先 `eth_getCode(address)`——有 code 走 1271，無 code 走 ecrecover；兩邊都要綁 **chainId + 正確 domain**（同 EIP-712）。
- 風險：假合約亂 return magic、簽名可重放（跨鏈／跨 dApp）、Safe 換 owner 後舊簽仲有冇效——產品要定「綁定時快照」定「每次提現即時驗」。

## 常見追問（連答案）

**Q：點解唔可以對合約地址做 ecrecover？**  
A：合約地址冇對應私鑰；簽係 owners／module 產嘅，要合約按自己規則驗證。硬 ecrecover 只會還原出某個 EOA，同合約地址對唔上。

**Q：CEX 提現白名單「簽名授權」要點設計？**  
A：建議 EIP-712 typed data（含 `userId`、`withdrawAddr`、`chainId`、`nonce`／deadline）；驗證時：EOA → ecrecover；合約 → 1271。`nonce` 防重放；上鏈前再查一次（Safe 可能已換 signer）。

**Q：同「訊息上寫 Sign in with Ethereum」有咩關係？**  
A：SIWE／好多 wallet-connect 流程底層仍係個人簽或 typed data；對 Smart Account 要支援 1271，否則 Safe 用戶登入／綁定會失敗。面試可提「檢測 code.length + 1271 fallback」。

## 小練習

**題：** 用戶用 Safe 地址綁 CEX。後端只做了 `ecrecover(EIP712Hash, sig)`，永遠失敗。你會點改？仲要防邊兩個坑？

**參考答案：**  
(1) `getCode(safe) != 0x` 時改調 `isValidSignature(hash, sig)`，檢查 magic `0x1626ba7e`；(2) 確認 hash 同 Safe 期望嘅是同一套（有啲 Safe 要對 `SafeMessage`／EIP-712 再包一層，唔係 raw digest）。防坑：跨 chain 重放（domain 帶 chainId／verifyingContract）；以及綁定後 owner 變更——提現授權應帶 deadline／每次即時 1271，唔好只信「綁定當日驗過一次」。
