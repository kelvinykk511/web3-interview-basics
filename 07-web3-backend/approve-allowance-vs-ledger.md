# ERC-20 approve/allowance vs 交易所內部帳本

## 人話
approve 好似簽一張「授權扣款」：我准合約 X 最多從我錢包拎 N 個 USDT。錢仲喺我度，只係對方有權拎。同 CEX 內部「凍結」唔同——allowance 唔鎖錢，餘額可以少過 allowance。

## 面試短答
- `approve(spender, amount)` 設額度；`transferFrom` 由 spender 調用扣款，扣 allowance。
- allowance ≠ 餘額凍結：帳本唔可以因 allowance 記「已預留」。
- 風險：無限授權（2^256-1）合約被黑 → 用戶資產被抽；釣魚騙 approve / permit 簽名（鏈下簽名，無 gas 提示更危險）。
- CEX 角度：熱錢包/歸集合約盡量唔對外部合約無限 approve；用 permit 時驗 deadline、nonce、spender。

## 常見追問（連答案）
1. **approve 競態？** 由 N 改 M，spender 可以搶先用晒 N 再用 M。做法：先設 0 再設 M，或用 increase/decreaseAllowance。USDT 本身要求先歸 0。
2. **permit (EIP-2612) 好處同坑？** 一個簽名代替 approve 交易，省一步；但用戶可能唔知自己簽咗授權，騙簽常見。
3. **充值識別會唔會被 transferFrom 影響？** 會：入帳係睇 Transfer event 嘅 to，唔係睇 tx.to；transferFrom 觸發嘅 Transfer 一樣要入帳。

## 小練習
題：用戶話「我冇轉帳，但 USDT 喺錢包冇咗」，客服點初步判斷？
答：查該地址 Transfer 事件，搵出 from=用戶 嘅記錄，睇 tx 調用者係咪某個 spender 合約（transferFrom）；再查佢歷史 Approval 事件/permit，大概率係早前授權咗惡意或被黑合約。建議用戶 revoke allowance（設 0）。
