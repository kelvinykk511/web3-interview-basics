# Robinhood Chain：對 CEX 上架／充提

> 銜接：你已經識 [EVM 兼容鏈接入差異](./evm-compatible-chain-integration-differences.md)、[L2 充值最終性](./l2-deposit-finality.md)、[L2 出金同 L1 data fee](./l2-withdrawal-fee-quote-l1-data-fee.md)、[LayerZero OFT vs CCTP](./layerzero-oft-vs-cctp-rebalancing.md)、[Chainlink CCIP](./chainlink-ccip-cex-treasury.md)、同 [ERC-7683 意圖橋](./erc7683-intent-bridge-cex.md)。Robinhood Chain 係 2026 年上線嘅獨立 Arbitrum L2，**唔係 Ethereum，亦唔係 Arbitrum One**。對 CEX 嚟講，最大陷阱係：軟確認很快，但經典橋出金有 7 日挑戰期；另外官方穩定幣係 USDG，唔好假設有同址嘅 Circle USDC。

官方（以 2026-11-25 文件為準）：[About Robinhood Chain](https://docs.robinhood.com/chain/)、[Connecting](https://docs.robinhood.com/chain/connecting/)、[Transaction Finality](https://docs.robinhood.com/chain/transaction-finality/)、[Bridging](https://docs.robinhood.com/chain/bridging/)、[Gas & Fees](https://docs.robinhood.com/chain/gas-and-fees/)、[Token Contracts](https://docs.robinhood.com/chain/contracts/)、[Protocol Contracts](https://docs.robinhood.com/chain/protocol-contracts/)、[Run a Full Node](https://docs.robinhood.com/chain/run-a-full-node/)。Robinhood 新聞室同 Arbitrum 官方網誌都寫 **Public Mainnet 喺 2026-07-01 推出**（[Robinhood Newsroom](https://robinhood.com/newsroom/robinhood-accelerates-global-expansion-robinhood-chain-mainnet-stock-tokens-agentic-trading/)、[Arbitrum Blog](https://blog.arbitrum.io/robinhood-chain-mainnet/)）。

## 人話（從你熟的 CEX）

Robinhood Chain 係一條 **permissionless、EVM 兼容** 嘅以太坊 L2，用 **Arbitrum Dedicated Blockchains／Nitro**，數據可用性走 **以太坊 blob**，原生 gas 係 **ETH**。佢嘅產品定位偏代幣化 RWA（股票／ETF 等 Stock Tokens），但鏈本身係開放網絡：任何人可以連 RPC、部署合約。所以上架要分開兩盤數——「這條鏈嘅 ETH／USDG／WETH 充提」同「某隻 Stock Token 係咪可以當現貨交易對」；後者係合規同產品決定，唔係單靠 EVM 兼容就自動開。

網絡參數（官方）：

| | Mainnet | Testnet |
| --- | --- | --- |
| Chain ID | **4663** | **46630** |
| Gas／Currency | ETH | ETH |
| 公共 RPC（限速，官方寫唔建議生產用） | `https://rpc.mainnet.chain.robinhood.com` | `https://rpc.testnet.chain.robinhood.com` |
| Explorer | robinhoodchain.blockscout.com | explorer.testnet.chain.robinhood.com |

官方推薦生產走 Alchemy 等供應商；另外有 Chainstack、QuickNode、Blockdaemon 等。自建節點要用 Nitro，而且必須有 **以太坊 L1 execution RPC + beacon（讀 blob）**。公開 sequencer feed：`wss://feed.mainnet.chain.robinhood.com`，延遲備份約 **500ms**（`wss://delayed-feed.mainnet.chain.robinhood.com`），官方建議兩條一齊接、用 sequence number 去重；公共 feed 係 best-effort，冇 uptime 保證。

最終性分三層（官方 Transaction Finality）：

| 階段 | 典型時延 | 保證 |
| --- | --- | --- |
| Soft confirmation（sequencer 接單、排序、執行並出 receipt） | 亞秒級 | 只係 sequencer 承諾；若之後打上 L1 嘅 batch 用咗唔同順序，仍然可被推翻 |
| Posted to Ethereum（batch 進 L1 Inbox） | 數分鐘 | 此後要再改順序，除非以太坊自己 reorg |
| Ethereum finality | 上 L1 之後約 13 分鐘 | 繼承以太坊最終性，官方稱為不可逆 |

官方自己都寫：日常 UX 可以用軟確認；**高價值或不可逆操作，要等已上 L1，甚至等以太坊最終性**。CEX 入可用應跟呢個分層，唔好把亞秒 receipt 當成大額入帳。

排序模型好關鍵：官方寫 Robinhood Chain 用 **先到先得（FCFS）**，順序以到達 sequencer 嘅時間為準，**唔能夠靠加更高手續費插隊**。呢點同以太坊主網「加 tip 搶塊」唔一樣。出金卡住，唔好第一時間用主網嗰套 gas bump 思維；要先查 sequencer 有冇接單、receipt／feed 有冇呢筆，再查係咪根本未到達 sequencer。

費用仍然分兩截，但用戶只付一筆：L2 執行費 + 把 calldata 發上以太坊嘅 **L1 data fee**（隨以太坊擁塞同交易大小變）。`eth_estimateGas`／錢包報價會包埋兩部分。熱錢包要備 ETH，唔係備「Robinhood token」。

充提橋要分官方經典橋同合作路線：

- **Canonical Arbitrum bridge（信任最小化預設）：** 以太坊 → Robinhood Chain 存款官方寫通常約 **10 分鐘**，用 retryable ticket。如果 L2 一腿失敗（例如 gas 不夠），資金唔會消失，可喺 **7 日內** 喺橋介面手動 redeem。原鏈成功 ≠ 目標鏈已入帳。
- **Robinhood Chain → 以太坊提款：** 三步——L2 發起、等 **7 日挑戰期**（Arbitrum 欺詐證明）、再喺 L1 claim（要付 L1 gas）。L2 發起成功 ≠ ETH 主網已到帳。
- **更快嘅合作路線：** LayerZero OFT／Stargate（官方舉例 WBTC、USDG 等 OFT）、Chainlink CCIP／Transporter、Relay／Across 等意圖橋、LiFi／0x 聚合。速度係分鐘甚至秒級，但結算模型同經典橋唔同：源鏈 lock／burn／open 成功，唔等於目標鏈已 mint／放行。呢度沿用你哋已有嘅 OFT／CCIP／意圖橋規則。

代幣合約：**唔好假設同以太坊同址。** 官方寫，橋過去嘅 ERC-20 喺 Robinhood Chain 嘅地址可以唔同，要用 L2 Gateway Router 嘅 `calculateL2TokenAddress`，或查 [Protocol Contracts](https://docs.robinhood.com/chain/protocol-contracts/)。官方 [Token Contracts](https://docs.robinhood.com/chain/contracts/) 截至查閱當日列出嘅 canonical 資產包括：

- WETH：`0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73`
- USDG：`0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168`（Paxos；官方 About 頁亦把 Paxos／USDG 列為穩定幣基礎設施）

官方合約頁**沒有**把 Circle USDC 列作 canonical token。上架或入帳唔好假設「有 USDC 同址」。Stock Tokens 必須用官方 registry 嗰個合約；**ticker 相同但地址不同，就唔係官方 Robinhood Stock Token**。Stock Token 係代幣化證券類產品，有司法管轄限制，唔能當成普通 ERC-20 現貨自動開充提。

L1 核心合約（主網，只供對帳／監控，**唔係用戶充值地址**）包括 Rollup `0x23A19d23e89166adedbDcB432518AB01e4272D94`、Sequencer Inbox `0xBd0D173EEb87D57A09521c24388a12789F33ba96`、Delayed Inbox `0x1A07cc4BD17E0118BdB54D70990D2158AbAD7a2D`。L2 Gateway Router 主網係 `0x1E324B9316138CA9a73F960213621AD1aaf01B89`。測試網合約多數唔同，必須用 `(chainId, contract)` 做資產鍵。

另外：鏈上有一流 ERC-4337 支持。如果將來掃 AA 充值，要沿用你哋已有嘅 UserOp／counterfactual 規則，唔好只掃普通 `tx.to`。

## 面試短答

1. **Robinhood Chain 係 2026-07-01 上線嘅獨立 Arbitrum Nitro L2，主網 chainId 4663，測試網 46630，gas 係 ETH，DA 走以太坊 blob。** 唔係 Ethereum，亦唔係 Arbitrum One（42161）。
2. **確認分三層：亞秒軟確認、數分鐘上 L1、再等約 13 分鐘繼承以太坊最終性。** 官方建議高價值／不可逆要等上 L1 或最終性；CEX 大額入帳唔好用 sequencer receipt。
3. **排序係 FCFS，加 tip 唔能插隊。** 出金卡住先查有冇被 sequencer 接單，唔好照搬主網 gas bump。
4. **經典橋：入金約 10 分鐘且係 retryable；出金要 7 日挑戰期再 L1 claim。** 源鏈成功 ≠ 目標鏈到帳。OFT／CCIP／意圖橋要分開對帳。
5. **官方列出嘅 canonical 資產包括 WETH 同 USDG，地址同以太坊唔一定相同。** 唔好假設 Circle USDC；Stock Token 要對 registry 地址，唔能只對 ticker。

## 常見追問（連答案）

**Q1：用戶展示亞秒 receipt，可唔可以入可用？**  
A：零售小額如果產品批准，最多當「已接單、未結算」。軟確認只係 sequencer 承諾，尚未繼承以太坊安全；官方亦寫高價值要等上 L1 或最終性。大額、出金對沖、不可逆操作等 `posted` 或 `finalized`。游標要記 tx hash／L2 塊 hash，唔好只數確認數。

**Q2：點解出金加咗好多 priority fee 都冇提前？**  
A：呢條鏈官方係 FCFS，順序睇到達 sequencer 嘅時間，唔係價高者得。先確認交易有冇被接單；若仍喺你哋本地 pending、未見於 feed／receipt，加 tip 唔會插隊。真正要處理嘅係連接、nonce、或 sequencer 狀況。

**Q3：用戶話已經從以太坊橋過去，但 Robinhood Chain 帳戶係 0，點查？**  
A：經典橋係 retryable。L1 交易成功只代表票據已開，L2 一腿可能因 gas 失敗，資金仲可以喺 7 日內 redeem。入帳只認 L2 實收（對應 token 合約嘅 Transfer 或 ETH 餘額），唔好用 L1 成功預入。同時核對 L2 token 地址係咪 `calculateL2TokenAddress` 嘅結果，而唔係以太坊原址。

**Q4：從 Robinhood Chain 提 ETH 回主網，L2 交易成功咗，幾時可以入用戶嘅 ERC-20／現貨帳？**  
A：唔可以即入。經典橋要等 7 日挑戰期，再等用戶或服務喺 L1 claim。應記「L2 已發起／挑戰期／L1 已 claim」三態。如果走 OFT、CCIP 或意圖橋，規則同經典橋唔同，但同樣係目標鏈實收先入帳。

**Q5：可唔可以用以太坊或 Arbitrum One 嘅 USDC／WETH 地址做入帳？Stock Token 點處理？**  
A：唔可以假設同址。官方 canonical WETH／USDG 已公布；USDC 唔在官方 token 列表，唔好當原生資產自動開。Stock Token 必須用官方 registry 合約，ticker 撞名唔算。呢類代幣化證券要獨立合規審查，唔好當普通 ERC-20 掃到就上架。資產鍵一律 `(4663, contract)`，同測試網 46630 分開。

**Q6：公共 RPC 同 sequencer feed 可唔可以直接做生產掃鏈？**  
A：官方寫公共 RPC 限速、唔建議生產。Feed 亦無 uptime 保證，應同時接 delayed backup（約滯後 500ms）並用 sequence number 去重。生產用供應商或自建 Nitro（要備 L1 EL + beacon）。

## 小練習（題目 + 參考答案）

**題目：** 組員打算開 Robinhood Chain 充提，方案係：(1) 複用 Arbitrum One（42161）掃鏈同 USDC／WETH 合約地址，確認數用「亞秒 receipt + 12 確認」；(2) 用戶以太坊經典橋交易成功就預入可用；(3) 用戶喺 Robinhood Chain 發起提回以太坊，L2 成功即喺主網現貨入帳；(4) 出金卡住就自動加倍 tip 重發；(5) 見到 ticker 係 `AAPL` 或 `USDC` 嘅 ERC-20 就當官方資產開充提。有咩問題？點改？

**參考答案：**

- **獨立鏈、獨立資產鍵。** chainId 係 4663，唔係 42161 亦唔係 1。WETH 用官方 `0x0Bd7…AD73`；USDG 用 `0x5fc5…d168`。唔假設 Circle USDC。測試網 46630 分開。入帳等官方最終性層級：大額等 batch 上 L1 或以太坊 finalized，唔用亞秒 receipt 加「12 確認」呢套主網啟發式。
- **經典橋只認 L2 實收。** L1 成功可能仲喺 retryable 待 redeem。反之，L2 發起提款後仲有 7 日挑戰期同 L1 claim，唔能當主網已到帳。
- **FCFS 下加倍 tip 唔會插隊，亂重發會卡 nonce。** 先查 sequencer receipt／feed；未接單先查廣播路徑。
- **Ticker 唔等於官方資產。** Stock Token 要對官方 registry 地址，並走獨立合規；USDC 未列 canonical 就唔好自動開。OFT／CCIP／意圖橋另外建流水，源鏈成功只記在途。
