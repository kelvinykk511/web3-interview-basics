# Monad：異步執行同最終性對 CEX 充提

> 銜接：快出塊≠最終性見 [Base Denim](./base-denim-200ms-blocks-cex.md)；EVM 兼容鏈差異見 [evm-compatible-chain-integration-differences.md](./evm-compatible-chain-integration-differences.md)；穩定幣合約要逐鏈核對見 [USDT0 上架](./usdt0-omnichain-listing-deposit.md)、上一篇 [Hyperliquid](./hyperliquid-hypercore-vs-hyperevm-cex.md)。今日講 Monad 主網（chainId **143**）同 Ethereum 最唔同嘅兩點：**共識同執行分開**，以及 **JSON-RPC 嘅 latest／safe／finalized 含義變咗**。

官方（以而家文件為準，唔好背舊 blog 數字）：[Current Facts](https://docs.monad.xyz/ai/current-facts)、[Network Information](https://docs.monad.xyz/developer-essentials/network-information)、[Block States](https://docs.monad.xyz/monad-arch/consensus/block-states)、[Asynchronous Execution](https://docs.monad.xyz/monad-arch/consensus/asynchronous-execution)、[Gas Pricing](https://docs.monad.xyz/developer-essentials/gas-pricing)、[Reserve Balance](https://docs.monad.xyz/developer-essentials/reserve-balance)、[Wallet integration](https://docs.monad.xyz/developer-essentials/wallet-developers)、[Tokens and Bridges](https://docs.monad.xyz/developer-essentials/network-information/tokens-and-bridges)。截至 2026-11-20，主網版本 `v0.16.5`／`MONAD_TEN`。

## 人話（從你熟的 CEX）

你喺 Ethereum 習慣：出塊時已經執行完，塊頭帶 state root；`latest` 雖然可能 reorg，但 receipt 至少係「呢個節點認為已經執行咗」。Monad 把 **排序（共識）** 同 **執行** 拆開：節點先對交易順序達成共識，**唔使先執行、亦唔喺提案帶最新 state root**。執行喺另一條稍慢嘅泳道跑。官方參數 `D`／`k` 而家主網同測試網都係 **3**：每個提案會帶 **3 塊之前** 嘅 merkle root，錯咗塊會被拒；`Verified`（執行結果被共識確認）大約係 `finalized - 3`。

區塊有四個本地狀態，對應 RPC 標籤時要小心：

| RPC 標籤 | Monad 狀態 | 含義（對 CEX） |
| --- | --- | --- |
| `latest` | Proposed | 先見到嘅頭，**尚未**最終確認，極少數情況可以唔入正典 |
| `safe` | Voted（投機最終性，約 1 個 300ms slot） | 超級多數已投票，仍然理論上可回退 |
| `finalized` | Finalized（約 2 個 slot，**600ms**） | 官方同 wallet 文件：入帳、橋、不可逆結算等呢個 |

官方性能數字（優先用呢頁，唔好用舊文）：出塊約 **300ms**（2026-09 主網均值約 302ms）、投機最終性 300ms、最終性 600ms、塊 gas 上限 150M、單筆 30M。因為共識只用延遲 3 塊嘅狀態睇 gas，協議加咗 **Reserve Balance**：`user_reserve_balance = 10 MON`。執行時，帳戶期末餘額（退款前）跌穿 10 MON 嘅 value spend 可能 revert（有例外）；剛從 0 餘額入帳嘅地址，要等資助交易舊咗 **D=3** 塊（文件寫大約再等 0.9 秒）先可以自己發交易。

Gas 同 Ethereum 有個會痛嘅分別：Monad **按 `gas_limit × price` 收費，唔係 `gas_used`**（否則有人可以報好大 limit、實際用好少，佔滿區塊卻少付，變成 DoS）。出金／歸集唔好再習慣性把 estimate 乘 2～5 倍。EIP-1559 兼容，最低 base fee 係 **100 MON-gwei**。官方公開 RPC 有速率限制，而且寫明 `rpc.monad.xyz` **唔應用於 production**。無 type-3 blob；7702 有額外限制（委託帳戶唔可以再把餘額降至 10 MON 以下；委託代碼唔可以 `CREATE`／`CREATE2`）。無全局 mempool。

代幣要逐鏈抄官方表：USDC（CCTP）`0x754704Bc059F8C67012fEd69BC8A327a5aafb603`；USDT0（LayerZero OFT）`0xe7cd86e13AC4309349F30B3435a9d337750fC82D`——**唔係** Plasma／HyperEVM 嗰個 `0xB8CE…5ebb`。WMON 係 `0x3bd359C1119dA7Da1D913D1C4D2B7c461115433A`。資產 key 仍然係 `(143, contract)`。

## 面試短答

1. **入帳等 `finalized`，唔好用 `latest` receipt。** Wallet 官方例子：見到 receipt 只代表呢個 RPC 已本地執行；要再等到 `finalized.number >= receipt.blockNumber`。`latest` 只係 Proposed。
2. **最終性 600ms ≠ 可以抄「確認數 1」就當 Ethereum 咁安心。** 300ms 出塊好快，`block.number + N` 嘅牆鐘會縮水；政策應對齊 `finalized`（同你們風險等級），唔好搬 32 個 Ethereum 確認，亦唔好把 `safe` 當成不可逆。
3. **出金 gas 按 limit 計費。** `gas_paid = gas_limit * price_per_gas`。寬估會直接燒 MON；用 Monad 自己嘅 estimate（文件示例可從約 7.5% buffer 調起），再按生產數據收斂。
4. **熱錢包要懂 10 MON 儲備同 D=3。** 剛入帳嘅空地址唔能即刻轉出；歸集／「gas 資助完即掃走」可能 revert 或入唔到塊。預留儲備同等待，唔好當普通 EVM。
5. **USDT0／USDC 合約要查 Monad 官方表，唔好複製其他鏈地址。** 呢點同 HyperEVM／Plasma「同址」相反，正好說明唯一可靠規則係 `(chainId, contract)`。

## 常見追問（連答案）

**Q1：共識都 finalized 咗，點解 state root 仲未 Verified？可唔可以先入帳？**  
A：Finalized 係對**交易順序**不可逆（除非硬分叉）。執行結果要等延遲 merkle root 被納入後續塊，先進入 Verified（大約 finalized 減 3 塊）。對 CEX 充值，receipt 加上 `finalized` 已經係官方建議嘅結算點；唔好用 `latest`／即時 WS 頭入可用。真正對帳、輕客戶端證明先需要 Verified 狀態。

**Q2：點解有啲「剛充 MON 就出金」會失敗？**  
A：空帳戶收到 MON 之後，自己發交易要等資助 tx 舊咗 `D=3` 塊。另外執行可能因為期末餘額跌穿 10 MON 而 revert。熱錢包要留儲備、資助同花費分塊，唔好同一輪把餘額掃到接近零（除非符合官方例外）。

**Q3：掃描器同 Ethereum 共用「latest + 12 確認」可唔可以？**  
A：唔可以。12 個 300ms 塊只係約 3.6 秒，而且 `latest` 起計本身就包含未最終確認嘅 Proposed 塊。改為跟 `finalized` 游標；getLogs 回看用塊數時按 300ms 換算，唔好當 12 秒一塊。Production 用付費／自建節點，唔好用官方公共 RPC。

## 小練習（題目 + 參考答案）

**題目：** Monad 上架 MON／USDC／USDT0。掃描器共用 Ethereum 邏輯：`latest` 見到 receipt 再加 3 個 confirmation 就入可用。出金 `eth_estimateGas` 之後把 gas limit 乘 5，跟住即刻把熱錢包 MON 掃到只剩手續費塵埃。USDT0 合約直接複製 HyperEVM 嘅 `0xB8CE…5ebb`。有咩問題？點改？

**參考答案：**

- **入帳對齊 `finalized`：** receipt 之後等到 finalized 高度覆蓋該塊。3 個 confirmation 從 latest 起計既短、又不最終。大額可再加你們自己的延遲，但基線係 finalized，不是 latest。
- **Gas 按 limit 收費：** 5 倍 padding 會真金白銀燒掉。用 Monad estimate + 小 buffer，監控 out-of-gas 再調。
- **儲備同 D=3：** 熱錢包保留至少 10 MON 量級嘅儲備，資助同轉出之間等 3 塊；唔好剛入帳就清空。
- **合約用 Monad 官方表：** USDC `0x754704Bc059F8C67012fEd69BC8A327a5aafb603`，USDT0 `0xe7cd86e13AC4309349F30B3435a9d337750fC82D`，資產 key `(143, contract)`。複製 HyperEVM／Plasma 地址會掃錯或上錯幣。
