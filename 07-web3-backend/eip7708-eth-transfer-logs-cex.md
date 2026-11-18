# EIP-7708：ETH 轉帳出 Transfer log（CEX 充值掃鏈）

> 銜接：而家原生 ETH 充值冇 `Transfer`、要掃 `tx.value`／internal，見 [native-eth-vs-erc20-deposit.md](./native-eth-vs-erc20-deposit.md)；validator withdrawals、fee recipient、SELFDESTRUCT 等非交易入帳見 [eth-non-tx-balance-changes.md](./eth-non-tx-balance-changes.md)；log 索引陷阱見 [event-log-indexing-pitfalls.md](./event-log-indexing-pitfalls.md)。Glamsterdam 分叉進度同 8037／8038 見 [glamsterdam-eip8037-8038-state-gas-cex.md](./glamsterdam-eip8037-8038-state-gas-cex.md)——**7708 同 8037／8038 一樣，截至 2026-11-18 都未上主網**。

官方說明：[EIP-7708](https://eips.ethereum.org/EIPS/eip-7708)（頁面標 Last Call，deadline 2026-11-01）、[EIP-7773 Hardfork Meta — Glamsterdam](https://eips.ethereum.org/EIPS/eip-7773)（7708 列喺 Scheduled for Inclusion）。Glamsterdam 主網日期未定；Hoodi 因 8037／8038 已延期。規格同列入名單都以 EIP 同 ACD 為準，唔好當主網已生效。

## 人話（從你熟的 CEX）

你而家掃 ERC-20 充值，係聽合約嘅 `Transfer(from, to, value)`；掃原生 ETH 就要另開一條路：睇交易 `value`、internal call，再加餘額差額兜底。原因好簡單——協議本身唔會為 ETH 轉帳出 event。面試都會問：合約錢包／Aggregator 一次呼叫入面轉 ETH 過嚟，單睇 `tx.to == 充值地址` 會漏。

EIP-7708 想把兩條路收斂：凡係**非零、轉去另一個帳戶**嘅 ETH 移動，協議都會自動出一條同 ERC-20 **相同 signature** 嘅 `Transfer` log。Emitter **唔係**某個 ETH token 合約，而係系統地址 `0xfffffffffffffffffffffffffffffffffffffffe`（`SYSTEM_ADDRESS`，同 EIP-4788 一類協議 log 同一思路）。涵蓋：

- 交易本身帶 `value`（呢條 log 會排喺該 tx 入面任何 EVM 自發 log **之前**）
- 非零 `CALL`（按呼叫發生次序）
- 非零 `CREATE`／`CREATE2` 給新合約嘅 endowment
- 非零 `SELFDESTRUCT` 轉去另一個帳戶

EIP **明確唔出 log** 嘅，仍然要舊方法兜：零值、revert／被回滾嘅內部轉帳、優先費俾 coinbase、base fee 銷毀、以及 validator withdrawals（唔掛喺 tx 上，而且區塊裡已經有現成欄位）。所以 7708 **唔等於**「以後 ETH 餘額變化全部都可以只聽 log」。

對 CEX：啟用之後，合約錢包、router 內部轉入嘅 ETH，可以同 ERC-20 一樣用 `getLogs` 捕到，早期交易所「合約入金要人工／延遲好耐」呢類問題會少好多。但如果你只認 `topic0 == Transfer`、唔分 `log.address`，系統 log 會被當成「不明代幣」，或者相反把 ERC-20 誤記成 ETH。分叉前嘅歷史塊亦唔會有呢啲 log，過渡期要雙軌。

## 面試短答

1. **7708 係 Glamsterdam 已排期列入嘅規格，不是主網現況。** 截至 2026-11-18，主網啟用日未定。答「ETH 而家都有 Transfer event」會被當場打臉。而家做 scanner 雙軌同回歸，啟用當日先切流量。
2. **認 log 要同時認 emitter。** `topic0` 同 ERC-20 `Transfer` 完全一樣；ETH 嘅 `log.address` 必須係 `SYSTEM_ADDRESS`，ERC-20 仍然係代幣合約。充值掃描：`address = SYSTEM_ADDRESS`、`topic0 = Transfer`、`topic2 = 充值地址`。冪等仍然用 `txHash + logIndex`（必要時加 chainId）。
3. **可以補到舊掃描器嘅盲區，但不能只靠它。** 合約錢包／internal `CALL`／`CREATE2` endowment 終於有統一 log；但 fee recipient、base fee 燒、validator withdrawals、零值、同帳戶轉帳、以及 reverted 內部轉帳都**冇** 7708 log。餘額差額對帳同 [非交易入帳](./eth-non-tx-balance-changes.md) 仍然要留。
4. **分叉邊界要雙軌。** 啟用前嘅塊繼續用 `value`／trace；啟用後先把系統 `Transfer` 當主路徑。唔好對歷史塊空等 log，亦唔好喺啟用後仍只掃 `tx.to + value`（會漏 internal）。
5. **最終性同 reorg 規則唔變。** 出咗 log ≠ 可以提前入可用。大額仍然等你哋既定嘅 `safe`／`finalized`／確認數。7708 唔加最壞情況 log 數量上限（EIP 寫 ETH 轉帳本身已比 `LOG3` 貴），但平均 log 量會升，容量同告警要一併評估。

## 常見追問（連答案）

**Q1：啟用之後，可唔可以停掃 `tx.value` 同 internal，只聽 Transfer？**  
A：唔可以一刀切。系統 `Transfer` 只覆蓋「非零、轉去另一帳戶、而且最終成功落地」嘅轉帳。Coinbase 優先費、validator withdrawals 仍然冇呢條 log。分叉前歷史更係完全冇。正確做法係：啟用後以系統 log 作 ETH 轉帳主路徑，非交易來源同餘額差額兜底保留，並做分叉邊界回歸。

**Q2：topic0 都係 Transfer，會唔會同 ERC-20 搞亂？**  
A：會，如果你只索引 topic、唔索引 emitter。系統 log 嘅 `address` 係 `0xff…fe`，唔係 WETH，亦唔係「原生 ETH 合約」。帳本 asset 仍然係 chain 上嘅 native ETH；唔好為 7708 另開一隻 ERC-20 asset_id。WETH 嘅 `Transfer` 仍然嚟自 WETH 合約，入帳做 WETH，唔做 ETH。

**Q3：SELFDESTRUCT、CREATE2、失敗嘅內層 CALL 呢？**  
A：非零且轉去**另一個帳戶**嘅 `SELFDESTRUCT`／`CREATE`／`CREATE2` endowment 會出 log（7708 依賴 EIP-8246，唔再處理「SELFDESTRUCT 燒俾自己」呢類舊邊緣）。內層 CALL 如果之後被 revert 回滾，規格列作負向案例：**唔出**（或唔應被當成最終入帳）。入帳只認最終成功、且已過確認水位嘅 receipt／log。

## 小練習（題目 + 參考答案）

**題目：** 主網宣布 7708 已啟動。掃描器為咗「統一」，改成：凡 `topic0 == Transfer` 且 `topic2` 係充值地址，就入一筆原生 ETH；同時關掉 `tx.value`／internal／餘額差額。另外把 fee recipient 熱錢包增加嘅 ETH，同 validator withdrawal，都當作用戶充值。有咩問題？點改？

**參考答案：**

- **必須過濾 emitter：** 只有 `log.address == SYSTEM_ADDRESS` 先係原生 ETH；代幣合約嘅 `Transfer` 入對應 ERC-20／WETH。否則會把 USDT／WETH 當成 ETH，或把系統 log 當成不明幣。
- **唔能關舊路徑一刀切：** 優先費、withdrawals、分叉前歷史、零值都冇 7708 log。fee recipient 同 validator withdrawal 本來就唔係用戶 `Transfer`，應走 [非交易入帳](./eth-non-tx-balance-changes.md) 嘅獨立來源同唯一鍵，禁止當充值。
- **internal／合約入金：** 7708 正正係補呢塊；啟用後應用系統 log 捕 CALL／CREATE2，而唔係繼續只掃 `tx.to`。
- **入帳水位不變：** receipt 有 log ≠ 最終性。保留 reorg rewind；冪等 `chainId + txHash + logIndex`。
- **上線節奏：** 主網日期未定就唔好當已生效；先雙軌對照，分叉當日先切主路徑，並監控 log 量同漏掃。
