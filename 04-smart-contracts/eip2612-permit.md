# EIP-2612 Permit（免 approve 交易嘅授權）

## 人話

昨日講 `approve` 要上鏈燒 gas，仲有「非零改額度」同無限授權坑。**Permit**（EIP-2612）畀用戶用 **鏈下簽名** 授權：一張 typed data 簽好，spender／router 喺同一筆（或隨後）交易入面叫 `permit`，合約核簽後直接改 `allowance`，用戶唔使先發一筆 approve tx。

CEX 類比：以前要用戶先打一隻「開 API 權限」工單上鏈；而家用戶簽一張 **離線授權書**，業務系統代為提交，少一次鏈上往返。

## 面試短答

- 標準接口：`permit(owner, spender, value, deadline, v, r, s)`；用 EIP-712 結構化簽名
- 合約內：驗 `deadline`、驗簽名對應 `owner`、跟 `nonce`（防重放）、再 `_approve`
- 好處：少一筆 approve gas、UX 更好（尤其 4337／meta-tx／router 打包）
- 範圍：要 token **實作** ERC-20 Permit（DAI 早期有類似、USDC／不少現代 token 有）；唔係所有 ERC-20 都支援
- 風險：簽名洩漏 = 授權被用；要管 `deadline`、spender、value；唔等於可以跳過對 spender 嘅信任評估

## 常見追問＋答案

Q：Permit 同普通 approve 最終狀態有冇分別？  
A：最終都係改 `allowance[owner][spender]`。分別喺 **點樣觸發**：approve 靠 owner 發 tx；permit 靠 owner 簽名 + 任何人（通常 spender）代交 tx。

Q：點解要 token 自己嘅 `nonces(owner)`？  
A：每用一次 permit 遞增，防止同一張簽名被重放多次加授權。同帳戶 tx nonce 係兩套概念——一個係帳戶發 tx 排序，一個係 permit 簽名防重放。

Q：CEX 熱錢包出金要唔要 permit？  
A：熱錢包自己簽 `transfer`／內部歸集通常唔使。Permit 常見於 **用戶錢包 ↔ DeFi router／聚合器**。若 CEX 做「代客授權＋兌換」產品，先要確認該 ERC-20 是否支援 2612，否則 fallback 兩步 approve。

Q：簽名過期同 value=max 點睇？  
A：`deadline` 過咗會 revert，降低長期暴露。`value=max` 仍然係無限授權語意，只係提交方式變咗；安全審查一樣要收緊 spender 同額度。

## 小練習

題：產品要做「一鍵授權並 swap」。鏈上 token 支援 EIP-2612。請寫出用戶側同後端／合約側最少步驟（含失敗點）。

參考答案：  
1）前端／後端組 EIP-712 typed data：`owner、spender=router、value、nonce=nonces(owner)、deadline`；  
2）用戶錢包簽出 `v,r,s`（唔上鏈）；  
3）呼叫 router 嘅 multicall／單筆：先 `token.permit(...)` 再 `transferFrom`＋swap；  
4）失敗點：token 無 permit → 要退回 approve；deadline 過期；nonce 已變（用戶同時簽過另一張）；spender 或 value 簽錯；permit 成功但 swap revert（allowance 已加，要設計清唔清授權）。CEX 對接時仲要校 `chainId` 同 token 合約地址，防簽錯鏈。
