# ERC-20 allowance / approve（授權陷阱）

## 人話

CEX 後端眼中：用戶「授權」某合約花佢錢包裡嘅代幣，好似畀一個 **API token 配額**——`approve(spender, amount)` 寫入 `allowance[owner][spender]`；之後 spender 先可以用 `transferFrom` 扣錢。

充幣入帳多數只掃 Transfer；但 **歸集、DeFi 對接、部分提現路徑、第三方託管** 會碰到 approve。面試常考：點解要兩步、無限授權風險、approve 競態。

## 面試短答

- 兩步模型：`approve` 改授權額度；真正轉帳係 spender 嘅 `transferFrom(from, to, value)`（要 `allowance >= value`）
- 查授權：`allowance(owner, spender)`（可用 `eth_call`）
- 常見模式：精準額度 vs `type(uint256).max` 無限授權（體驗好、風險大）
- 舊版 ERC-20 陷阱：部分實作要求「非零→零→新值」先改额度，否則 revert；另有 front-running 改額度攻擊敘事
- 安全：用戶／熱錢包唔應隨便永久授權未知 spender；CEX 自有熱錢包通常直接 `transfer`，少用對外 approve

## 常見追問＋答案

Q：點解轉 USDT 去 Uniswap 要先 approve？  
A：DEX 合約唔係 token owner；佢要 `transferFrom` 用戶餘額，必須先有 allowance。CEX 內部歸集若熱錢包自己簽 `transfer`，就唔使 approve 自己。

Q：無限授權（max uint）有咩問題？  
A：一次授權後，spender（或被盜／惡意升級嘅合約）可反覆 `transferFrom` 抽走餘額，直到用戶 `approve(0)` 或轉走資產。面試要講清「便利 vs 攻擊面」。

Q：approve 從 N 改到 M，點解有人話要先改 0？  
A：部分舊 token（含早期 USDT 風格）`approve` 喺已有非零 allowance 時直接改會 revert；業務要做 `approve(0)` 再 `approve(M)`。另有經典「改額度 front-run」討論：攻擊者趁新旧额度窗口多用一次。現代多推 `increaseAllowance`／`permit`（EIP-2612）或直接用 0→M 流程。

Q：CEX 掃充幣要唔要理 approve？  
A：標準充幣入帳睇 `Transfer` 到充值地址即可；approve 本身唔轉資產。但風控／歸集／對外 DeFi 對接要監控異常授權同 spender 黑名單。
