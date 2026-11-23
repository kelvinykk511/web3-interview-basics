# Circle Arc：原生 USDC Gas 對 CEX 上架／充提

> 銜接：你已經識 [EIP-7708 ETH Transfer log](./eip7708-eth-transfer-logs-cex.md)、[USDC CCTP 調撥](./usdc-cctp-cross-chain-rebalancing.md)、[Circle Gateway](./circle-gateway-unified-usdc-balance.md)、同 [Tempo 穩定幣手續費](./tempo-tip20-virtual-address-cex.md)。Arc 係 Circle 嘅穩定幣 L1：Gas 同原生餘額都係 USDC，冇獨立嘅波動性 gas token。對 CEX 嚟講，陷阱係「睇落好似普通 EVM，但其實 `eth_getBalance`、Transfer decimals、掃鏈 emitter 同最終性模型全部同以太坊唔一樣」。

官方（以而家文件為準）：[Connect to Arc](https://docs.arc.io/integrate/connect-to-arc)、[Stablecoin native model](https://docs.arc.io/arc/concepts/stablecoin-native-model)、[USDC system events](https://docs.arc.io/arc/references/usdc-system-events)、[Detect deposits](https://docs.arc.io/integrate/exchanges/deposits)、[Infrastructure](https://docs.arc.io/integrate/infrastructure)、[Gas and fees](https://docs.arc.io/arc/references/gas-and-fees)、[Contract addresses](https://docs.arc.io/arc/references/contract-addresses)、[Deterministic finality](https://docs.arc.io/arc/concepts/deterministic-finality)。主網已於 **2026-09-16** 上線。

| 項目 | 主網 | 測試網 |
| --- | --- | --- |
| chainId | `5042` | `5042002` |
| RPC | `https://rpc.mainnet.arc.io` | `https://rpc.testnet.arc.io` |
| 出塊 | 約 0.5 秒 | 約 0.48 秒 |
| Gas／原生資產 | USDC | USDC |
| 共識 | 許可制驗證者 + Malachite BFT | 同左 |
| EVM | Osaka | Osaka |
| CCTP domain | `26` | `26`（合約地址同主網不同） |

## 人話（從你熟的 CEX）

用戶入嘅「幣」同你付嘅 gas，其實係同一筆 USDC 餘額，但有兩張面孔：

| 介面 | 用法 | Decimals |
| --- | --- | --- |
| 原生（`eth_getBalance`、`tx.value`／`msg.value`） | Gas、原生轉帳 | **18** |
| ERC-20（`0x3600000000000000000000000000000000000000` 嘅 `balanceOf`／`transfer`） | approve、應用層轉帳 | **6** |

呢個地址喺主網同測試網 **一樣**，所以資產 key 必須帶 chainId：`(5042, USDC)` 對 `(5042002, USDC)`。兩邊唔係兩種代幣，只係同一餘額嘅兩個視圖：ERC-20 會截走最後 12 位，細過 1×10⁻⁶ USDC 嘅原生塵埃，`balanceOf` 睇唔到，但仍然可以當 gas 花。

掃充值唔可以只聽 ERC-20 合約。官方（Arc 對 EIP-7708 嘅實現）規定：所有明確嘅 USDC 流動——原生轉帳、ERC-20 轉帳、mint／burn——都會由系統地址 `0xffffFFFfFFffffffffffffffFfFFFfffFFFfFFfE` 出一條 **18 decimals** 嘅 `Transfer`。純原生轉帳 **唔會** 喺 `0x3600…0000` 出 log；ERC-20 `transfer()` 則兩邊都會出。所以官方教交易所：**只濾系統 emitter**，用 `txHash + logIndex` 冪等，金額用 18 位入帳，唔好先截成 6 位。Self-transfer 同零金額冇 log；gas 費同出塊獎勵亦唔出 `Transfer`（gas 用 receipt 計，獎勵睇 `block.miner`）。主網由 genesis 就用呢套；測試網 Zero5 分叉前先至係 `0x1800…0000` 嘅舊 `NativeCoin*` event，返歷史先要處理。

最終性官方寫係確定性、一秒內、**已出塊就唔會 reorg**，交易所指引係 **1 個確認即可入帳**，唔使再疊 6 個確認。但共識係 **許可制 PoA + BFT**，唔係以太坊嗰種大眾驗證者集：協議承諾冇 reorg，營運上仍要監控出塊／停擺。另外亞秒出塊可以令連續幾塊共用同一個 `block.timestamp`，游標同對帳要用塊號，唔好用時間窗當唯一鍵。`PREVRANDAO` 恆為 `0`；冇 blob 交易。

出金／歸集記住三條：`maxFeePerGas` 最少 **20 Gwei**（低過呢個可能直接唔入塊、亦冇 error receipt）；熱錢包要備 USDC 付 gas，唔存在「再轉一筆 ETH 做油費」；blocklist 轉帳會 revert——交易可能已入塊並扣咗 gas，但狀態回滾，一定要查 `receipt.status`，唔好當廣播成功就扣用戶帳。EIP-3009 relayer 代交嘅充值，`tx.from` 係 relayer，要用系統 `Transfer` 嘅 `from` 做歸因。官方歸集示例用原生 `value`（18 位），gasLimit 示例係 `65,000`，唔好硬編碼 21,000 就當一定夠。

EURC、USYC、bridged WETH、cirBTC 都係獨立資產，有自己嘅合約同 decimals（EURC 主網 `0xbEf5f6d5…21c1`，測試網地址不同；USYC 仲有資格限制）。唔好把 Bridged WETH 當成「呢條鏈嘅原生 ETH」。CCTP domain `26` 只係編號，合約主網／測試網不同；源鏈 burn ≠ Arc 已到帳。Gateway 合約喺 Arc 亦有部署，但同「鏈上原生 USDC 餘額」唔好混成一個入帳來源，呢點同 11-15 講過嘅 Gateway 模型一致。

## 面試短答

1. **Gas 同原生餘額都係 USDC，冇獨立 ETH。** `eth_getBalance` 係 18 位 USDC，唔好標成 ETH。
2. **同一餘額、兩套小數。** 原生 18、ERC-20 6；主網／測試網 USDC 合約同址，資產 key 一定帶 `5042`／`5042002`。
3. **充值只掃系統 emitter `0xffff…fffe`（18 位）。** 只掃 `0x3600…0000` 會漏原生轉帳；兩邊都掃會重複入帳。
4. **官方：已入塊即最終，1 個確認可入帳。** 許可制驗證者，仍要監控停擺；唔好用 timestamp 當唯一鍵。
5. **出金：USDC 付 gas，`maxFeePerGas ≥ 20 Gwei`，檢查 receipt／blocklist，歸因用 event `from`。**

## 常見追問（連答案）

**Q1：可唔可以沿用 ERC-20 USDC 掃描器，金額一律當 6 decimals？**  
A：唔得。原生轉帳唔經 ERC-20 合約，會漏帳。系統 log 係 18 位，截成 6 位會把少於 1×10⁻⁶ USDC 嘅塵埃吞掉，而呢啲塵埃仍然可以付 gas，對帳會唔平。正確係濾系統 emitter、以 18 位入帳；ERC-20 `balanceOf` 只作對照，唔好同 18 位直接加減。

**Q2：確定性最終性係咪等於可以完全唔做 reorg 回滾，亦唔使理驗證者集？**  
A：官方對 integrator 嘅指引係：已 committed 嘅塊係永久嘅，1 確認即可入帳，唔使疊確認。呢個係 BFT／許可制模型下嘅協議承諾，唔係「再等 32 個 slot」。但許可制意味住出塊依賴獲准驗證者；停擺或驗證者事故要有暫停充提同告警，而唔係假設永遠有下一個塊。

**Q3：用戶話「我轉咗 ETH／USDC 過去」，或者用咗 EIP-3009，點處理？**  
A：鏈上冇原生 ETH，只有 bridged WETH（主網 `0x128cC470…84EDB`，18 位），要獨立資產、獨立掃 Transfer。USDC 無論原生定 ERC-20 都入同一個 USDC 帳，但要以系統 event 為準。EIP-3009 被 relayer 提交時，`tx.from` 係 relayer，要用系統 `Transfer.from` 認存款人。CCTP 入金等 Arc 側 mint，唔好用源鏈 `DepositForBurn` 預入。

## 小練習（題目 + 參考答案）

**題目：** 上架 Arc USDC。組員打算：(1) 把 `eth_getBalance` 當成 ETH，再另開一個 ERC-20 USDC（`0x3600…0000`）帳戶，兩邊都掃 Transfer 入帳；(2) 主網同測試網合約同址，就共用一個資產碼；(3) 沿用以太坊 12 個確認；(4) 出金用 21,000 gas、低於 20 Gwei 嘅 `maxFeePerGas`，成功廣播就扣帳，並用 `tx.from` 做充值歸因。有咩問題？點改？

**參考答案：**

- **只有一個 USDC 餘額。** `eth_getBalance` 係 18 位 USDC。只濾系統 emitter `0xffff…fffe` 入帳；唔好把 ERC-20 log 再入一次，亦唔好標成 ETH。
- **資產 key 帶鏈：** 主網 `(5042, 原生 USDC)`，測試網 `(5042002, 同一個 0x3600… 地址)`。EURC／WETH／USYC 分開。
- **已入塊且 `status=1` 就可按官方指引入可用**（1 確認）；保留出塊中斷告警，唔使疊以太坊式確認數。游標用塊號。
- **出金熱錢包備 USDC；`maxFeePerGas ≥ 20 Gwei`；gasLimit 唔好死用 21,000（官方 sweep 示例 65,000）。** 低於下限可能唔入塊。Blocklist revert 要認 `status=0`。歸因用系統 `Transfer.from`。歸集 `value` 用 18 位原生單位。
