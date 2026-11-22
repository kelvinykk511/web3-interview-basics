# Starknet／StarkGate：對 CEX 上架、充值同出金

> 銜接：近排接過 MegaETH、Monad、Hyperliquid、Plasma 呢啲「EVM 兼容但確認規則唔一樣」嘅鏈。Starknet **唔係 EVM**：帳戶同地址都係 felt，ETH 喺 L2 係代幣合約而唔係 native `value`，手續費用 **STRK**。官方橋係 StarkGate。資產 key 仍然要帶鏈：`(SN_MAIN, l2_token)`，唔好同以太坊 ETH／USDC 混庫。

官方（以而家文件為準）：[Chain information](https://docs.starknet.io/learn/cheatsheets/chain-info)、[Transactions](https://docs.starknet.io/learn/protocol/transactions)、[StarkGate](https://docs.starknet.io/learn/protocol/starkgate)、[Fees](https://docs.starknet.io/learn/protocol/fees)、[Transaction fields / chain ID](https://docs.starknet.io/learn/cheatsheets/transactions-reference)、[官方橋代幣表](https://github.com/starknet-io/starknet-addresses/blob/master/bridged_tokens/mainnet.json)。截至 **2026-11-22**，主網同 Sepolia 都係 Starknet **0.14.4**（Sierra 1.9.0）。

## 人話（從你熟的 CEX）

接入一條 EVM 鏈，你會用 20-byte 地址、`eth_getLogs` 掃 Transfer、native 幣當 gas、確認數或 `safe`／`finalized`。Starknet 呢幾樣都要改口。

鏈 ID 唔係 1、4326 呢種小數。官方寫：chain ID 係常數字串嘅 big-endian ASCII 整數。主網 `SN_MAIN` = `0x534e5f4d41494e`，測試網 `SN_SEPOLIA` = `0x534e5f5345504f4c4941`。地址係 felt252，長度同校驗都唔係 EIP-55 嗰套 40 個 hex；**唔好把用戶嘅以太坊地址直接當成 Starknet 充值地址。**

L2 上連 ETH 都係代幣合約。官方主網（同 Sepolia 合約地址相同，所以更加要靠 chain ID 分開）：

| 資產 | L2 合約 | decimals |
| --- | --- | --- |
| STRK | `0x04718f5a0fc34cc1af16a1cdee98ffb20c31f5cd61d6ab07201858f4287c938d` | 18 |
| ETH | `0x049d36570d4e46f48e99674bd3fcc84644ddd6b96f7c741b1562b82f9e004dc7` | 18 |
| USDC | `0x053c91253bc9682c04929ca02ed00b3e423f6710d2ee7e0d5ebb06f3ecf368a8` | 6 |
| USDT | `0x068f5c6a61780768455de69077e07e89787839bf8166decfbf92b645209c0fb8` | 6 |

STRK／ETH 呢兩個地址喺主網同 Sepolia **一樣**，資產 key 必須帶 `SN_MAIN` 或 `SN_SEPOLIA`。官方表入面 L1 ETH 用咗佔位地址 `0x0000…4548`（ASCII `ETH`），代表嘅係 **L1 原生 ETH**，唔係用戶會持有嘅 L1 ERC-20。L2 ETH／STRK／USDC 先係合約。其他代幣以 [bridged_tokens/mainnet.json](https://github.com/starknet-io/starknet-addresses/blob/master/bridged_tokens/mainnet.json) 為準；部份舊幣仲有 legacy 專用 L1 橋，唔好全部當成同一個 TokenBridge。

手續費由 Starknet 0.13.0 起用 STRK 喺 L2 收取；0.14.0 起只接受按三種資源報價嘅 v3 交易：L2 gas、L1 gas、L1 data gas，價格以 fri 計（1 fri = 10⁻¹⁸ STRK）。費用係 OS 注入一筆 STRK transfer（付款人 → sequencer），而家 **唔燃燒**。熱錢包出金要備 **STRK**，唔好以為備 L2 ETH 就付到 gas。區塊預確認最長約 9.5 秒封塊（mempool 空同冇執行中交易時建議約 2 秒），每塊最多 500 筆交易。

最終性（0.14.0 起）同 EVM 嘅 `latest`／`safe`／`finalized` 唔共用名：

| 狀態 | 意思 | 對 CEX |
| --- | --- | --- |
| `RECEIVED` | 某一個 full node 已交去 sequencer | 換另一個節點查可能「hash not found」，唔好當確認 |
| `CANDIDATE` | sequencer 已寫入，但未執行 | 未有 receipt |
| `PRE_CONFIRMED` | sequencer 已執行並寫咗 receipt | 只係預確認 |
| `ACCEPTED_ON_L2` | 已入被 L2 共識最終化嘅塊 | 仲未上到以太坊 |
| `ACCEPTED_ON_L1` | 以太坊上嘅 Starknet 狀態高度已覆蓋呢個塊 | 入可用／大額結算對齊呢個 |

執行狀態另計：`SUCCEEDED`／`REVERTED`。被 revert 嘅交易仍會加 nonce、仍會收手續費，驗證階段或收費階段甚至可能仲有 event；執行階段嘅狀態、L1 message 同 event 會撤回。唔好處到一條 Transfer 就當業務入帳成功。

用戶從以太坊經 StarkGate 存入，官方生命周期係：

1. L1 叫 `deposit`：鎖定資金、出 `Deposit` event（帶 L1／L2 地址同金額）、發 L1→L2 message。呢段時間 L2 側可以仍係 `NOT_RECEIVED`，因為 sequencer 要等足夠 L1 確認。
2. L2 橋 `handle_deposit` 校驗來源後先 mint。到呢步先係 `ACCEPTED_ON_L2`。
3. Prover 提交 L1 state update，訊息喺 Core contract 被清除，先成為已證明嘅 Starknet 狀態。

所以 **L1 `Deposit` event ≠ L2 已到帳**，更唔等於可提現。`depositWithMessage` 仲要目標合約 `on_receive` 返回 true，否則 L1 handler 失敗，要用專用取消流程收回。一般存款如果遲遲未出現在 L2，存款人可申請取消，大約 **五日起** 先能 `depositReclaim`；在 reclaim 之前，存款仍有機會被處理。

從 L2 提回以太坊：L2 `initiate_token_withdraw` 會 **burn** 並發 L2→L1 message；prover 把 state update 交上 L1 後，訊息記喺 Core contract（主網 `0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4`），之後 **任何人** 都可以喺 L1 橋叫 `withdraw`，收款地址以訊息為準。**L2 burn ≠ L1 已入帳。** 預設無限額，但緊急時可以把單日提現限制設成受影響代幣 TVL 嘅 5%。交易所自己出金到 L1，要等 L1 真正 `withdraw` 成功先從「鏈上在途」改為「已到冷／熱錢包」。

帳戶係原生 AA（合約帳戶）。未部署都可以先收到代幣（counterfactual），但 `DEPLOY_ACCOUNT` 通常要預先有 STRK 先付到部署費。掃鏈唔好直接搬 EVM 嘅 20-byte topic；Starknet event 用 felt 地址，`uint256` 金額常見係兩個 felt（低位／高位）。

## 面試短答

1. **入可用對齊 `ACCEPTED_ON_L1`，唔好用 `PRE_CONFIRMED`／L1 `Deposit` event。** `ACCEPTED_ON_L2` 只係 L2 共識，仲未證明上以太坊。
2. **資產 key 係 `(SN_MAIN, l2_token)`。** STRK／ETH 合約喺主網同 Sepolia 同址；L2 ETH ≠ L1 原生 ETH；L2 USDC／USDT 用官方表，唔好抄 EVM 地址。
3. **Gas 係 STRK（fri），唔係 L2 ETH。** 0.14.0 起要 v3、三種 resource bounds。熱錢包備 STRK。
4. **StarkGate 出金：L2 burn 之後仲要等 L1 state update，再有人（或你們自己）call L1 `withdraw`。** 燒咗唔等於 L1 已到。
5. **地址同 event 都唔係 EVM 那套。** 唔好把 0x 以太坊地址當 L2 充值地址；revert 交易都可能有手續費 Transfer。

## 常見追問（連答案）

**Q1：點解唔喺 `ACCEPTED_ON_L2` 就入可用？預確認好快喎。**  
A：`PRE_CONFIRMED` 只係 sequencer receipt；`ACCEPTED_ON_L2` 係 L2 共識最終化，但錢喺橋兩邊嘅最終保證，要等 state update 被以太坊接受（`ACCEPTED_ON_L1`）。L1 入金尤其如是：L1 鎖咗之後，L2 mint 都仲要等 L1 確認同證明。要做「已看見」可以用更早狀態，但可交易／可提現餘額對齊 `ACCEPTED_ON_L1`。

**Q2：用戶用自己嘅以太坊地址作 Starknet 充值地址，或者你把 L1 USDC 合約抄去 L2 掃，會點？**  
A：L2 地址係 felt，唔等於 20-byte EOA。抄錯就掃唔到、亦入錯帳。USDC／USDT／ETH 都有獨立 L2 合約，見官方 `bridged_tokens` 表。STRK／ETH 再加一層陷阱：主網同 Sepolia 合約相同，唔帶 chain ID 會把測試網資金當成主網。

**Q3：L2 顯示已 withdraw／已 burn，點解 L1 熱錢包仲未收到？**  
A：官方流程第三步係 permissionless 嘅 L1 `withdraw`。訊息已寫上 Core contract 之後，要有一筆 L1 交易先把代幣從橋轉去收款地址。監察要分開「L2 已 burn」「L1 訊息已記錄」「L1 withdraw 已成功」。緊急期間仲可能開 5% TVL／日提現上限，出金要能解釋排隊而唔係當交易失敗重發。

## 小練習（題目 + 參考答案）

**題目：** 上架 Starknet 嘅 ETH、STRK、USDC。充值地址直接用用戶以太坊地址；掃描器見到 L1 StarkGate `Deposit` 就入可用。L2 掃 Transfer 時把 Sepolia 同主網嘅 ETH 合約當成同一個資產（因為地址一樣），gas 用 L2 ETH 報價。用戶從 L2 提回交易所 L1 地址，一見 L2 burn 就加 L1 ETH 可用。有咩問題？點改？

**參考答案：**

- **發 Starknet 專用 felt 地址（原生 AA），唔好用 L1 地址充數。** 未部署都可以收幣，但出金熱錢包要有 STRK 付 v3 費用。
- **入可用等該筆 L2 鑄幣／轉帳 `ACCEPTED_ON_L1`。** L1 `Deposit` 只係鎖倉＋訊息；其間狀態可以係 `NOT_RECEIVED`。`PRE_CONFIRMED`／`ACCEPTED_ON_L2` 只可作「已看見」。
- **資產 key 帶鏈：** 主網 ETH 係 `(SN_MAIN, 0x049d3657…4dc7)`，Sepolia 用同一個合約地址但係另一個資產。USDC 用官方 L2 `0x053c9125…68a8`（6 decimals），唔好抄 L1 `0xA0b8…`。
- **出金備 STRK，費用用三種 resource 嘅 v3 報價。** L2 ETH 係資產，唔係 gas。
- **L2→L1 出金：** 等 Core contract 記到訊息，並且 L1 `withdraw` 成功，先把 L1 ETH 入可用。Burn 只係在途。對帳用官方 Core `0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4` 同官方橋表，唔好假設所有幣共用同一個 legacy 橋。
