# MegaETH：迷你塊 vs EVM 塊，對 CEX 充提／上架

> 銜接：昨日 [Monad 異步執行／最終性](./monad-async-execution-cex-finality.md)、[Plasma USDT0](./plasma-usdt0-cex-deposit.md)、[Hyperliquid](./hyperliquid-hypercore-vs-hyperevm-cex.md) 已反覆提醒同一 USDT0 合約會出現喺 Plasma（9745）、HyperEVM（999）同 MegaETH（**4326**），資產 key 必須 `(chainId, contract)`。今日把 MegaETH 本身講清楚：佢係 **OP Stack L2**，但有兩種塊，`latest` 嘅意思同一般 Ethereum／OP 鏈唔一樣。

官方（以而家文件為準）：[Connect](https://docs.megaeth.com/user-guide/connect)、[Mini-Blocks](https://docs.megaeth.com/mini-block)、[Architecture](https://docs.megaeth.com/architecture)、[Read from MegaETH](https://docs.megaeth.com/developer-docs/overview-2)、[Developer FAQ](https://docs.megaeth.com/developer-docs/faq)、[Contracts & Tokens](https://docs.megaeth.com/developer-docs/overview-1/contracts)、[Get Funds / Bridge](https://docs.megaeth.com/user-guide/bridge)、[Dual Gas Model](https://docs.megaeth.com/spec/megaevm/dual-gas-model)、[`eth_gasPrice`](https://docs.megaeth.com/developer-docs/rpc/reference/eth_gasprice)、[`eth_chainId`](https://docs.megaeth.com/developer-docs/rpc/reference/eth_chainid)。截至 2026-11-21 嘅公開參數如下。

## 人話（從你熟的 CEX）

你接入一條「EVM 兼容 L2」通常會假設：`eth_blockNumber` 就係確認游標、`latest` receipt 再加 N 個 confirmation、出金 gas 最少 21000、gas 代幣就係嗰條鏈嘅 ticker。MegaETH 呢幾樣都要改口。

主網 chainId **4326**（`0x10e6`），測試網 6343（`0x18c7`）。**Gas 係 ETH（18 decimals），唔係 MEGA。** MEGA 係協議代幣 `0x28B7E77f82B25B95953825F1E3eA0E36c1c29861`，同 gas 分開入庫。官方公共 RPC：`https://mainnet.megaeth.com/rpc`，WebSocket `wss://mainnet.megaeth.com/ws`（有速率限制；文件寫而家未支援自建 RPC）。

MegaETH 同時出兩種塊：

| | Mini-block（迷你塊） | EVM block |
| --- | --- | --- |
| 間隔 | 約 **10ms** | 約 **1 秒** |
| 用途 | Realtime API，低延遲預確認 | 標準 JSON-RPC、錢包、現有 indexer |
| 狀態根 | 冇 | 有 |
| 時間戳精度 | 微秒 | 1 秒；合約入面 `block.timestamp` 係 **EVM 塊** 時間 |

每筆交易剛好出現在一個迷你塊同埋一個 EVM 塊。Sequencer 大約每 10ms 封一個迷你塊，即時流去 RPC；大約每秒先把期間嘅迷你塊封成一個標準 EVM 塊。一筆交易可以喺 EVM 塊未封好之前，就喺 realtime receipt 見到 `blockNumber`，但 `blockHash` 仍然係 `null`——呢個只代表已入迷你塊、被 sequencer 預確認，**未入 EVM 塊**。

JSON-RPC 標籤要小心：

| 標籤 | 數據來源 | 對 CEX |
| --- | --- | --- |
| `latest`／`pending` | **串流狀態**，跟最新迷你塊 | 快，只係 sequencer 預確認 |
| `safe` | 最新被視為 safe 嘅 **EVM 塊** | 仲未到 L1 最終性 |
| `finalized` | 最新 **finalized EVM 塊** | 入可用／大額結算對齊呢個 |
| 十六進制塊號 | 歷史 **EVM 塊** 狀態 | 標準掃鏈游標用 EVM 塊高 |

官方寫明：迷你塊頭上嘅 sequencer 簽名，只證明 sequencer **承諾**咗內容；如果之後封出嚟嘅 EVM 塊同簽名矛盾，可以舉證行為不當。但簽名**本身唔等於最終性**——完整最終性仍要等包含它嘅 EVM 塊被提交並喺 **Ethereum L1 finalized**。數據可用性走 **EigenDA** 證書，再由 OP Stack batcher 交上 L1；爭議用 Kailua（RISC-Zero 上嘅 ZK fraud proof），唔係標準多輪互動 bisection。FAQ 亦寫：L1 reorg 有機會令 MegaETH 錨點失效，從而回滾已被預確認嘅塊（機會取決於 sequencer 選錨有幾進取，總體應該好低）。所以預確認／`latest` 唔能當不可逆入帳。

`uptime.megaeth.com` 顯示嘅高度係 **迷你塊**，唔係 EVM `block.number`。確認數、getLogs 回看、游標都要跟 EVM 塊（約 1 秒一塊），唔好把迷你塊高度當成「已確認 10 萬塊」。

Gas 係 **雙維度**：compute gas（繼承嘅 EVM 計算）＋ storage gas（狀態／日誌／calldata 負擔）。兩者都從 `gas_limit` 扣。內在 storage gas 係 **39,000**，加 21,000 compute，所以官方 FAQ 寫原生轉帳最低約 **60,000 gas**，唔好再寫死 21000。當前費用政策下 `eth_gasPrice` 返回 `0xf4240`（**0.001 gwei**）；Connect 頁寫 EIP-1559 嘅 base fee 調節「實際上停咗」。單筆同每塊 gas 上限都係 **100 億**（10,000,000,000）。每個帳戶 txpool 最多 500 筆 pending。

代幣同樣要拆：USDT0 官方 `0xB8CE59FC3717ada4C02eaDF9682A9e934F625ebb`（6 decimals），同 Plasma／HyperEVM **同址**，必須帶 chainId 4326。官方主網表有 **USDM** `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7`（MegaETH 原生穩定幣），**冇**把 Circle USDC 列作獨立主網資產；官方橋頁嘅 USDm Bridge 係把 USDC **1:1 兌換成 USDM**，唔能當成「鏈上 USDC」。USDe／sUSDe 都有獨立合約，資產 key 仍然係 `(4326, contract)`。從以太坊官方 Standard Bridge 入 ETH，要轉去 L1 `0x0CA3A2FBC3D770b578223FBB6b062fa875a2eE75`，而且要等 **以太坊交易 finalized** 之後先會喺 MegaETH 出現——源鏈 lock 唔等於目標鏈可花費。

## 面試短答

1. **入帳對齊 `finalized` EVM 塊，唔好用迷你塊／`latest`。** `latest`／`pending` 讀嘅係 10ms 串流預確認。Realtime receipt 有 `blockNumber` 但 `blockHash` 為空，只係未入 EVM 塊，唔係最終性。
2. **10ms 出塊 ≠ 最終性升級。** Sequencer 簽名可以驗證預確認，但官方明確話最終性仍要等 L1。亦唔好搬 32 個 Ethereum 確認：32 個 EVM 塊只係約 32 秒，同 L1 finalized 唔係同一件事。
3. **掃鏈游標用 EVM `block.number`，唔好用迷你塊高度。** 監控面板嘅高度係迷你塊；合約 `block.timestamp` 亦係 EVM 塊（整秒）。
4. **Gas 代幣係 ETH，最低轉帳約 60,000，唔係 21000；MEGA 係另一個資產。** 當前 gas price 約 0.001 gwei。公共 RPC 有限速，文件寫暫不支援自建節點，生產要用官方允許嘅 managed RPC。
5. **資產 key 仍係 `(4326, contract)`。** USDT0 同址唔等於同一條鏈；USDM ≠ USDC；官方 Standard Bridge 入 ETH 要等 L1 finalized，唔好源鏈 lock 就加 MegaETH 可用。

## 常見追問（連答案）

**Q1：迷你塊都有 sequencer 簽名，點解仲要等 L1 finalized？**  
A：簽名只證明「呢個 sequencer 承諾咗呢個結果」，方便低延遲展示同舉證不作爲。最終性同數據可用性、L1 爭議窗口綁埋。官方一句：有效簽名 ≠ 最終性。CEX 入可用跟 `finalized`；要做「已看見、未入帳」可以用 mini-block／`latest`，但唔好把錢變成可提現、可交易餘額。

**Q2：L1 會唔會把已經預確認嘅充值推倒？**  
A：FAQ 寫：每個 MegaETH 塊錨住一個 L1 塊，L1 reorg 可以令錨失效並回滾對應嘅 MegaETH 塊。機會應該好低，但預確認路徑必須有回滾處理（未 credited 就撤銷 seen／confirming）。已 credited 之後先 reorg 就要走你們既定嘅凍結／人工／倒扣政策，所以大額更應該一開始就等 `finalized`。

**Q3：出金把 gas limit 當 21000，或者把 MEGA 當 gas，有咩後果？**  
A：雙 gas 之下原生轉帳內在成本已係 21,000 compute ＋ 39,000 storage。寫死 21000 會出唔到。MEGA 唔係 gas，熱錢包要備 **ETH**。fee 好平（0.001 gwei）唔代表可以亂填超大 limit 去做狀態爆炸式合約；storage gas 就係為呢樣而設。

## 小練習（題目 + 參考答案）

**題目：** 上架 MegaETH 嘅 ETH、MEGA、USDT0、「USDC」。掃描器訂閱 Realtime 迷你塊，見到 receipt（有時 `blockHash` 為空）就當 1 個 confirmation；再用 `uptime.megaeth.com` 嘅高度加 32 入可用。出金 gas 寫死 21000，熱錢包只備 MEGA。USDT0 合約複製 Plasma 地址但資產 key 唔帶 chainId；把官方橋兌換到嘅 USDM 當住 Circle USDC 入帳。從以太坊 Standard Bridge 一見到 L1 lock 就加 MegaETH ETH 可用。有咩問題？點改？

**參考答案：**

- **入帳等 `finalized`：** 迷你塊／空 `blockHash` 只係預確認。32 個「確認」如果跟錯迷你塊高度會極短；就算跟 EVM 塊，32 秒亦唔等於 L1 最終性。游標同 getLogs 用 EVM 塊高。
- **熱錢包備 ETH，gas ≥ 約 60,000：** 唔好寫死 21000，亦唔好把 MEGA 當 gas。
- **USDT0 用 `(4326, 0xB8CE59FC3717ada4C02eaDF9682A9e934F625ebb)`：** 同址唔能同 Plasma／HyperEVM 混庫。
- **USDM ≠ USDC：** 官方主網表係 USDM `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7`；USDm Bridge 係兌換，要獨立資產同獨立風控。
- **官方橋入 ETH：** L1 合約 `0x0CA3A2FBC3D770b578223FBB6b062fa875a2eE75`，等以太坊 finalized，再等 MegaETH 出現並對齊 `finalized` 先入可用。源鏈 lock ≠ 目標鏈可花。
