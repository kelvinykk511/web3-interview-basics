# ERC-4337 Counterfactual 地址：未部署都可以收幣

## 人話（從你熟的 CEX）

CEX 派「專屬充值地址」好似派子帳戶號碼。4337 Smart Account 好多用 **CREATE2**：地址 = `hash(factory, salt, initCode)`，**合約未部署都已經計得到**。用戶可以先把幣打去呢個「未來帳戶」；第一次 UserOp 帶 `initCode`，EntryPoint／工廠先部署，之後先用得到帳戶邏輯。

對後端嘅含義：**`getCode(address)` 係空 ≠ 唔可以收／唔可以出**；但亦都 **≠ 呢個地址喺每條鏈都係同一個人**。

## 面試短答

- **Counterfactual**：工廠 + salt + 初始化參數預先決定地址；未部署時 `extcodesize==0`，仍可作 ETH／ERC-20 收款地址。
- **首次部署**：UserOp 的 `initCode`（factory + calldata）喺驗證階段由 EntryPoint 部署；之後 `sender` 就有 code。
- **CEX 充值**：可以收；scanner 唔需要等帳戶部署完先入帳（錢已經喺嗰個地址）。
- **CEX 出金去 AA 地址**：可以出到未部署地址；提醒用戶 **只限該鏈**；已部署合約收 ETH 要 `estimateGas`，唔寫死 21000。
- **跨鏈陷阱**：同一「邏輯帳戶」喺另一條鏈未必用同一 factory／salt／初始化；地址撞唔到或落到別人控制——出金白名單要綁 `(chainId, address)`。
- **驗簽**：未部署帳戶登入／綁定可能走 **ERC-6492**（簽名包裹 factory 資料），唔可以只 ecrecover。

## 常見追問（連答案）

**Q1：未部署地址被人轉錯 ERC-20 進去，用戶之後 deploy 唔到／salt 錯會點？**  
A：幣已經喺嗰個地址。若用戶永遠用錯參數部署唔到「自己控制」嘅實作，呢啲幣可能永久鎖死（視乎之後有冇人能用正確 initCode 部署同一 CREATE2 地址）。客服／風控要當高風險找回案；產品應喺派地址時寫死 factory＋參數，並顯示「未部署亦可收、首次使用會自動部署」。

**Q2：點解唔好用 `getCode==0` 判斷「一定係 EOA、出金 gas=21000」？**  
A：Counterfactual AA、未部署合約、甚至某些預編譯邊緣情況都會令 code 為空但仍唔係「普通 EOA 轉賬」語義。出金：先 `eth_estimateGas`；對明顯合約（有 code 或已知 factory 地址簿）走合約收款路徑（見 [`eth-withdrawal-to-contract-wallet.md`](./eth-withdrawal-to-contract-wallet.md)）。

**Q3：充值專屬地址用 AA CREATE2，同用 EOA HD 派址點揀？**  
A：EOA HD：熱錢包可控、歸集簡單、簽名路徑清晰。AA CREATE2：適合「用戶自管 Smart Account、CEX 只係顯示／監控地址」或「產品本身就係 AA 錢包」。CEX 若仍要歸集，要麼用戶帳戶有 sweep 權限（session key／guardian），要麼仍用傳統「充值入熱錢包 EOA／合約」模型。面試講清：**派址模型決定歸集同責任邊界**，唔好混用又假設可以任意掃走用戶 AA 裏嘅錢。

## 小練習（題目 + 參考答案）

**題目**：產品要為 AA 用戶預先顯示「即將部署」嘅充值地址，並允許未部署就收款。設計派址同出金校驗；用戶之後喺 Arbitrum 用「同一個」地址要求出金，你批唔批？

**參考答案**：
- 派址：後端保存 `(userId, chainId, factory, salt, initCodeHash, predictedAddress)`；UI 標明鏈同「合約可能尚未部署」。充值 indexer 按普通地址監控，唔依賴部署事件。
- 出金校驗：目標地址綁定必須係 **同一 chainId** 白名單項；跨鏈即使地址字串相同，**預設拒絕或要求重新驗證擁有權**（6492／1271／挑戰簽名），因為 CREATE2 參數跨鏈唔保證同一控制者。
- 可選：出金前 `getCode`；若已部署，記錄 implementation／EIP-1271 支援，方便客服。Gas 用 estimate + buffer + 上限。
- 回歸：未部署收款 → credit；錯誤 chain 同址出金 → 拒；部署後 1271 綁定提現地址 → 過。
