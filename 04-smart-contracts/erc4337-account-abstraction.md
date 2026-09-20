# ERC-4337 Account Abstraction（UserOperation／Paymaster）

## 人話（從你熟的 CEX）

普通帳戶（EOA）＝「有私鑰先簽得 tx」。合約錢包想自己驗證規則（多簽、社交恢復、session key），舊世界要靠別個 EOA 幫佢 `eth_sendRawTransaction`。  
**ERC-4337** 唔改協議共識：用戶交一份 **UserOperation** 去 mempool／Bundler，Bundler 打包 dial `EntryPoint`，由 EntryPoint 驗簽、扣 gas、再 call 你個 Smart Account。CEX 會撞到：用戶用 AA 錢包充值、用 Paymaster 代付 gas、或者產品要支援「無原生幣帳戶」。

## 面試短答

- **UserOperation**：唔係傳統 tx；欄位含 `sender`、`nonce`、`callData`、`signature`、gas 上限、可選 `paymasterAndData`。
- **EntryPoint**：單一（或少數）系統合約；負責 `validateUserOp` → 扣費 → `execute`；失敗要可 revert 到驗證邊界。
- **Bundler**：類似「專門打包 UserOp 嘅 relayer」；有自己的 mempool 同模擬規則（防 DoS）。
- **Paymaster**：可代付 gas（sponsor／ERC-20 付 gas）；CEX 若自研贊助要防「無限贊助」同簽名授權範圍。
- 同 EIP-1271：Smart Account 驗簽好多時走 1271／自訂 `validateUserOp`；綁定／登入仍要支援合約錢包路徑。

## 常見追問（連答案）

**Q：點解 4337「唔使硬分叉」？**  
A：共識層仍然只認普通 tx。UserOp 由 Bundler 包成一筆普通 tx call EntryPoint；驗證邏輯喺合約層，唔喺共識。

**Q：CEX 充值掃描要改咩？**  
A：入帳仍睇 **Transfer／native 到帳地址**（多數 Smart Account 仍係一個合約地址）。唔好假設「只有 EOA 會收幣」。提現若用戶要你打去 AA 地址：當普通合約收款即可；若對方用 Paymaster／session key，唔影響你出帳路徑。真正要改嘅係「簽名綁定／登入」同「檢測 getCode + 1271／4337」。

**Q：Paymaster 最大風險？**  
A：被刷贊助（惡意 UserOp 耗光 gas 預算）、驗證同扣費不一致（模擬通過、上鏈失敗）、以及 paymaster 私鑰／後端授權被盜。面試答：quota、per-user cap、模擬（`eth_estimateUserOperationGas`／EntryPoint simulate）、同簽名帶 deadline／nonce。

## 小練習

**題：** 產品話「畀用戶用 Smart Account 登入 CEX，而且首次綁定唔使戶口有 ETH」。你會點串 4337／1271／Paymaster？邊兩條防線必加？

**參考答案：**  
登入／綁定：有 code → EIP-1271（或專用 SIWE＋1271）驗簽，唔走純 ecrecover。Gas：首次 onboarding 若要上鏈（例如 deploy account／寫 session），可用受控 Paymaster 贊助，**只允許白名單 selector／工廠**。防線：(1) Paymaster 對 `sender`／`callData` 做 allowlist + 日額度；(2) 綁定訊息帶 `userId`、`chainId`、`nonce`／deadline，防重放同跨 dApp 重用簽名。
