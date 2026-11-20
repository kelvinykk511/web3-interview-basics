# Hyperliquid：HyperCore vs HyperEVM 對 CEX 上架／充提

> 銜接：昨日 [Plasma USDT0](./plasma-usdt0-cex-deposit.md) 已提醒同一 USDT0 合約地址會出現喺 HyperEVM（chainId 999）、Plasma（9745）同 MegaETH（4326）；資產 key 必須 `(chainId, contract)`，見 [USDT0 上架](./usdt0-omnichain-listing-deposit.md)、[錯鏈錯幣](./wrong-chain-token-recovery.md)。今日講清楚 **「Hyperliquid」其實有兩套帳本**，以及 HYPE／USDC／USDT0 不能合成一個庫存。

官方：[HyperEVM](https://hyperliquid.gitbook.io/Hyperliquid-docs/for-developers/hyperevm)、[HyperCore ↔ HyperEVM transfers](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/hypercore-less-than-greater-than-hyperevm-transfers)、[Dual-block architecture](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/dual-block-architecture)、[Interaction timings](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interaction-timings)、[USDC](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/usdc)、[How to use the HyperEVM](https://hyperliquid.gitbook.io/hyperliquid-docs/onboarding/how-to-use-the-hyperevm)。規格以官方文件為準。

## 人話（從你熟的 CEX）

產品講「上架 Hyperliquid」，好容易當成一條普通 EVM 鏈。其實同一條鏈分成兩邊：

- **HyperCore**：永續／現貨訂單簿同相關餘額，**唔係** EVM。充提、轉帳走官方 L1 action／API，不是 `eth_sendRawTransaction`。
- **HyperEVM**：標準 EVM，Mainnet chainId **999**（Testnet 998），gas 係 **HYPE**（EVM 側 18 decimals），官方 RPC `https://rpc.hyperliquid.xyz/evm`。官方寫明呢個 endpoint **冇** websocket JSON-RPC。硬分叉基準係 Cancun **冇 blobs**；EIP-1559 開住，base fee 燒掉，而且因為用 HyperBFT，**priority fee 都會燒**，記入零地址嘅 EVM 餘額。

兩邊**唔會自動互通**。Core 現貨要同 EVM 上嘅 ERC-20 先「link」再轉換。系統地址規則：一般 token 係首字節 `0x20`、其餘 byte 用 token index 大端編碼；例如 USDC 喺 Core 係 token index 0，系統地址 `0x2000…0000`。HYPE 例外，固定 `0x2222…2222`：由 Core 轉去 EVM 會變成 **native HYPE**；由 EVM 轉返 Core，就係對呢個地址轉 `value`，合約 `receive()` 會出 `Received(user, amount)`（`user` 係 `msg.sender`）。EVM→Core 嘅 ERC-20 係轉去對應系統地址，以 linked 合約嘅 `Transfer` 為準；Core→EVM 則由系統交易代為 `transfer(recipient, amount)`。官方明確講：**唔好盲目假設 Core 同 EVM spot 1:1 可互換**——linked 合約可以係任意 bytecode，多出嚟嘅 wei decimals 如果金額唔整除，碎額會被 burn（保證 <1 Wei）。

時序：Core→EVM 會排到下一個 HyperEVM 塊；EVM→Core 喺同一個 L1 塊、EVM 塊起完之後即刻處理。HyperEVM 仲有 **雙區塊**：快塊約每 **1 秒**、gas 上限 3M；慢塊約每 **1 分鐘**、30M。兩種塊共用一條遞增嘅 EVM `block.number`，mempool 只接受每個地址接下來 **8 個 nonce**，超過一日會被清。

代幣更要拆開：HyperEVM 上 Circle USDC 官方指向 `0xb88339cb7199b77e23db6e890353e22632ba630f`；USDT0 官方合約 `0xB8CE59FC3717ada4C02eaDF9682A9e934F625ebb`（同 Plasma／MegaETH 同址，但 chainId 唔同）。WHYPE 係官方不可變包裝合約 `0x555…5`（即 `0x5555555555555555555555555555555555555555`），同 native HYPE 唔係同一個 asset_id。官方亦提醒：有啲 CEX 只支援 **Core 現貨 HYPE**，唔支援 HyperEVM；出金前要確認網絡。USDC 官方而家優先 CCTP 原生鑄造；舊 Arbitrum 橋已標 deprecated，唔好再把「Hyperliquid USDC 出金 = 一定去 Arbitrum」寫死。

## 面試短答

1. **「Hyperliquid」唔等於一條帳本。** 上架至少要分清 HyperCore 同 HyperEVM（999）。只開一邊就要喺產品同地址文案寫死；用戶把 EVM HYPE 打去只收 Core 嘅地址（或相反），會卡住或當錯網絡。
2. **資產 key 仍然係 `(鏈, 合約或賬本類型)`。** Core HYPE ≠ EVM native HYPE ≠ WHYPE；Circle USDC ≠ USDT0。USDT0 合約可以同其他鏈相同，但必須帶 chainId 999，唔好把 Plasma／MegaETH 嘅 Transfer 算入 HyperEVM。
3. **Core 唔能用 EVM 掃描器充數。** 掃 chain 999 嘅 native value／ERC-20 `Transfer` 只覆蓋 HyperEVM。Core 入帳要走官方 ledger／info／websocket 等 L1 資料，出金亦唔好套 EVM nonce／gas。
4. **系統地址同零地址唔係用戶充值目的地。** 轉去 `0x2222…`／`0x2000…` 係 Core↔EVM 轉換；priority fee 燒到零地址都唔能入帳。反之，系統交易把 token `transfer` 到你哋充值地址，先至可能係 EVM 側入帳。
5. **確認數唔好抄 Ethereum，亦唔好假設 1 個 block.number = 1 秒。** 塊繼承 HyperBFT（官方講 one-block finality），但慢塊每分鐘先出、同樣佔一個遞增塊號。官方 RPC 無 WS，要輪詢或改用支援訂閱嘅節點。大額用你哋既定最終性政策。

## 常見追問（連答案）

**Q1：用戶話「我已經充咗 HYPE 上 Hyperliquid」，點解我哋掃唔到？**  
A：先問清楚係 HyperCore 現貨還是 HyperEVM（錢包有冇加 chainId 999）。只掃 EVM 會漏掉 Core 側嘅 spot 轉帳；只聽 Core ledger 會漏掉 EVM native／ERC-20。兩邊要分開地址、分開掃描、分開客服話術。

**Q2：EVM 上收到 USDT0 同 USDC，可唔可以合成一個「USD 穩定幣」庫存？**  
A：唔可以。合約唔同、發行同跨鏈路徑都唔同（USDT0 走 OFT；USDC 官方優先 CCTP）。歸集、出金、對帳一律按 `asset_id = (999, contract)`。WHYPE 同 HYPE 亦唔能 1:1 合併，要有明確包裝／拆包裝流程。

**Q3：雙區塊同系統轉帳會點影響掃鏈？**  
A：`block.number` 前進唔等於牆鐘每秒一塊。getLogs 回看秒數如果按「1 塊 = 1 秒」會估錯。Core→EVM 入帳可能要等多一個 HyperEVM 塊先出現 `Transfer`；見到系統地址轉入先入帳，唔好喺 Core 側已扣就預入 EVM 可用。Linked 合約要做上架核對，因為協議唔保證 `transfer` 行為。

## 小練習（題目 + 參考答案）

**題目：** 產品只寫「HYPE／USDC（Hyperliquid）」。熱錢包只生成一個 0x 地址，掃描器只跟 Ethereum 一樣等 32 個 confirmation，而且只用官方 `rpc.hyperliquid.xyz/evm` 開 websocket 訂 `Transfer`。有人把 Plasma 上同一 USDT0 地址嘅轉帳當成 Hyperliquid 入帳；另一人把 EVM HYPE 轉去 `0x2222…2222` 之後，系統把「轉去系統地址」當成用戶出金成功。有咩問題？點改？

**參考答案：**

- **拆產品同資產：** 分開 HyperCore 同 HyperEVM（999）；HYPE／WHYPE／USDC／USDT0 各有 asset_id。地址頁寫明收邊一邊、邊一個合約。
- **掃描對應帳本：** EVM 用 chainId 999 嘅 native／`Transfer`（官方 RPC 無 WS，改輪詢或其它節點）；Core 走 L1 API／ledger。32 個 Ethereum 式確認不適用；跟執行塊同 HyperBFT 最終性政策。
- **資產 key 帶 chainId：** Plasma／MegaETH 即使合約字面相同，都唔能入 HyperEVM 帳。
- **系統地址係轉換，唔係出金目的地：** 轉去 `0x2222…`／token 系統地址代表去咗另一邊帳本，要等對側實際入帳再改狀態，唔好當已付到用戶。零地址因 priority fee 增加嘅餘額當燃燒，唔入充值。
