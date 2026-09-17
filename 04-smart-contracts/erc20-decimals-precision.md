# ERC-20 Decimals／精度 — CEX 帳本點對鏈上單位

## 人話

CEX 帳本常見用固定精度（例如 8 位、18 位、或內部最小單位）。  
鏈上 ERC-20 **每個合約自己宣告 `decimals()`**：USDC 多數 6、DAI／WETH 多數 18、有啲山寨仲有 0 或 9。  
`transfer`／`balanceOf` 用嘅係**整數最小單位**（唔係「人睇嘅 1.5 USDC」），所以後端一定要：`鏈上 amount = 人話數量 × 10^decimals`。

搞錯 decimals＝多送／少計／對唔到帳，係充提事故高發區。

## 面試短答

ERC-20 數量係 `uint256` 最小單位；`decimals` 只係顯示／換算約定，**唔參與轉帳邏輯本身**。  
CEX 資產主鍵除咗 `chainId × contract`，入帳換算要綁定該合約嘅 decimals（啟動時讀一次＋快取，異常時告警）。  
對帳：鏈上 `balanceOf(hot)` 同內部 ledger 要用同一 decimals 折算；跨 token 加總必須先轉正規化單位（例如全轉 18 位或全轉 USD）。

## 常見追問（連答案）

**Q: 點解唔可以假設全部 token 都係 18 decimals？**  
A: 主網穩定幣好多 6（USDC/USDT 視鏈而定）、WBTC 常 8。假設 18 會令顯示錯 10^(12) 倍級，提幣直接資金事故。

**Q: `decimals()` 係 view，會唔會被惡意假報？**  
A: 惡意合約可以對 `decimals()` 回謊，但真正轉帳仍跟 `balance`／`transfer` 整數。CEX 只上架白名單 token，decimals 以**上架審核＋官方文檔＋鏈上讀取交叉驗證**為準；運行中若讀數變更要當事故。

**Q: 同手續費、最小提幣額點配合？**  
A: 最小提幣、fee 都要落喺該 token 最小單位嘅整數格；用戶輸入「1.234567 USDC」要按 6 位截斷／四捨五入策略寫死（銀行家？向下？），同鏈上實際 `uint` 一致，避免 ledger 有碎步但鏈上發唔出。

**Q: 內部用 BigDecimal，鏈上用 uint，點防精準度 bug？**  
A: 邊界用整數最小單位做 source of truth；顯示層先轉小數。Java 用 `BigInteger` 存鏈上 amount，唔好用 `double`。入帳：先確認 decimals，再 `amount.multiply(TEN.pow(decimals))`。

## 小練習

**題：** 用戶提 1.5 USDC（假設該鏈 USDC `decimals=6`）。熱錢包合約 `transfer` 嘅 `_value` 應係幾多？若工程師誤當 18 decimals 會變成點？

**參考答案：**  
正確：`1.5 × 10^6 = 1_500_000`。  
誤當 18：會編成 `1.5 × 10^18`，鏈上會嘗試轉「天文數量」——通常 `balance` 不足 revert；若係低 decimals 錯反向，可能只轉極小額令用戶以為未到帳。預防：上架配置表強制 decimals、單測用真實 mainnet／fork 讀 `decimals()`、出金前 simulate／eth_call。
