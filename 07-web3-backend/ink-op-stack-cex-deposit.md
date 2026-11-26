# Ink：Kraken 嘅 OP Stack L2，對 CEX 上架／充提

> 銜接：你已經識 [EVM 兼容鏈接入差異](./evm-compatible-chain-integration-differences.md)、[L2 充值最終性](./l2-deposit-finality.md)、[Base Denim 200ms 出塊](./base-denim-200ms-blocks-cex.md)、[OP Superchain Interop](./op-superchain-interop-cex.md)、[USDT0](./usdt0-omnichain-listing-deposit.md)、同 [USDC CCTP](./usdc-cctp-cross-chain-rebalancing.md)。Ink 係 Kraken 推出、但**對其他交易所嚟講只係另一條獨立 OP Stack L2**——唔係 Ethereum、唔係 OP Mainnet、亦唔係 Base。最大陷阱：1 秒密封塊之外仲有約 200ms 嘅 subblocks 預確認；以及官方列表上嘅 USDC.e 同 Circle 原生 USDC 唔係同一個資產。

官方（以 2026-11-26 文件為準）：[Network Information](https://docs.inkonchain.com/general/network-information)、[About](https://docs.inkonchain.com/general/about)、[RPC／Subblocks](https://docs.inkonchain.com/tools/rpc)、[FAQ](https://docs.inkonchain.com/faq)、[Bridges](https://docs.inkonchain.com/tools/bridges)、[Fees](https://docs.inkonchain.com/build/transaction-fees)、[L1／L2 合約](https://docs.inkonchain.com/useful-information/contracts)、[Token Contracts](https://docs.inkonchain.com/useful-information/ink-token-contracts)、[Superchain registry `ink.toml`](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/ink.toml)。公開主網：[Kraken Blog 2024-12-18](https://blog.kraken.com/news/ink-mainnet-launch)。原生 USDC／CCTP V2：[Circle 2025-09-25](https://www.circle.com/blog/now-available-usdc-cctp-v2-on-ink)、[USDC 地址](https://developers.circle.com/stablecoins/usdc-contract-addresses)、[CCTP domains](https://developers.circle.com/cctp/references/contract-addresses)。階段同出金窗口：[ETHBEAT Ink](https://ethbeat.org/layer2s/projects/ink)（寫 Stage 1 自 2025-01-22）。

## 人話（從你熟的 CEX）

Ink 係一條 **EVM 兼容、OP Stack、Superchain** 嘅以太坊 L2，「Unleashed by Kraken」，定位偏 DeFi。Gas／原生貨幣係 **ETH**，數據可用性走 **以太坊 blob**（registry：`data_availability_type = "eth-da"`）。對你哋嚟講，Kraken 已支援 Ink 充提，只代表**對手所**接咗呢條鏈，唔代表你可以複用佢哋嘅充值地址、合約配置，或者把用戶「從 Kraken 提去 Ink」當成已經入咗你哋帳。

網絡參數（官方）：

| | Mainnet | Testnet（Ink Sepolia） |
| --- | --- | --- |
| Chain ID | **57073** | **763373** |
| Gas／Currency | ETH | ETH |
| 出塊時間（密封塊） | **1 秒**（About 寫 Day 1 已係 1s；sub-second sealed block「即將」來到，唔好當已經係） | 同系列測試網 |
| 公共 RPC 例子 | `https://rpc-gel.inkonchain.com`、`https://rpc-qnd.inkonchain.com` 等 | `https://rpc-gel-sepolia.inkonchain.com` 等 |
| Explorer | explorer.inkonchain.com | explorer-sepolia.inkonchain.com |
| Subblocks 流 | `wss://subblocks.inkonchain.com/ws` | `wss://subblocks-sepolia.inkonchain.com/ws` |

官方 About 寫 1 秒出塊；Superchain registry 嘅 `block_time = 1` 對得上。另外，官方 RPC 頁把 **Subblocks**（前稱 Flashblocks）定義為 sequencer 喺密封完整塊之前，大約每 **200ms** 推出嘅部分塊，讓應用提早見到交易結果。呢個只係**預確認**：信任假設喺 sequencer，可以喺 sequencer 重組 unsafe 塊嘅同一條件下被撤回。流式 payload 入面 `state_root` 同 `block_hash` **永遠係 0**，唔能當權威。若要用 `eth_call`／`eth_getBalance(pending)` 讀預確認狀態，要找支援 subblocks 嘅 RPC。呢點同 Base Denim 唔好混：Base 已經把密封塊本身改到約 200ms；**Ink 嘅密封塊仍然係 1 秒**，200ms 嘅係預確認流。

確認層級建議：

- **Subblock／pending 預確認：** 亞秒級 UX，可被撤回。CEX 最多當「sequencer 話接咗」，唔好入可用。
- **1 秒密封 L2 塊（unsafe／latest）：** 已封塊，但仍受 sequencer 同之後 L1 派生影響。零售小額若產品批准，都要清楚風險。
- **安全／最終性：** 要等派生到 L1、再繼承以太坊最終性。大額、出金對沖、不可逆操作跟呢一層，唔好數「12 個 L2 確認」呢套主網啟發式。

官方 FAQ 寫：**Ink 暫時冇自建 bridge portal**，用戶走合作橋（Across、Brid.gg、Bungee、Gelato、Rhino.fi、Superbridge、Reservoir 等）。但 L1 仍然有標準 OP Stack 橋合約，**唔等於沒有經典橋結算**：

- L1 Standard Bridge（以太坊主網）：`0x88FF1e5b602916615391F55854588EFcBB7663f0`
- OptimismPortal（以太坊主網）：`0x5d66C1782664115999C47c9fA5cd031f495D3e4F`
- L2 Standard Bridge（預部署）：`0x4200000000000000000000000000000000000010`

經典 L2→L1 出金要喺 L1 prove 再 finalize。ETHBEAT 寫 Ink 已係 **Stage 1**（里程碑 2025-01-22），interactive fraud proof 嘅挑戰期 **3 日 12 小時**，再加 **3 日 12 小時** execution delay，合共約 7 日先能喺 L1 取回。L2 發起成功 ≠ 主網已到帳。Sequencer 故障時，用戶可經 L1 強制送交易，最多有 **12 小時** 延遲。ETHBEAT 亦寫 upgrade 冇用戶退出窗口（Security Council／Foundation 可即時升級）——呢點上架時要知，但唔能單憑呢句拒絕或接受，要放入鏈風險評估。

代幣合約是高頻陷阱。官方 Token Contracts 頁（寫明仍在補充）列出例如：

- WETH9：`0x4200000000000000000000000000000000000006`（OP Stack 預部署）
- USDC.e：`0xF1815bd50389c46847f0Bda824eC8da914045D14`（橋接版本，**唔係** Circle 發行嘅原生 USDC）
- USDT0：`0x0200C29006150606B650577BBE7B6248F58470c1`

Circle 官方則另列 **Ink 原生 USDC**：`0x2D270e6886d130D724215A266106e6832161EAEd`（測試網 `0xFabab97dCE620294D2B0b0e46C68964e326300Ac`，唔好同主網 USDC.e 混淆）。CCTP V2 自 2025-09-25 可透過 domain **21** 進出，TokenMessengerV2 地址 `0x28b5a0e9C621a5BadaA536219b3a228C8168cf5d`。源鏈 burn 成功 ≠ Ink 已 mint 入帳；入帳只認目標鏈實收。資產鍵一律 `(57073, contract)`，**唔好複用** Ethereum／OP Mainnet（`0x0b2C…Ff85`）／Base（`0x8335…2913`）嘅 USDC 地址。

費用仍係 L2 執行費 + 發上以太坊嘅 L1 security／data fee；熱錢包要備 ETH，唔係備「Ink token」。FAQ 寫暫時沒有 TGE／官方代幣資訊，唔好假設有原生 gas token 要上架。

Superchain 成員身分 **唔等於** 統一 ETH 餘額。Registry 為 Ink 標咗 `lagoon_time = 2027-01-11 16:00:01 UTC`（香港時間 2027-01-12 00:00:01），但官方首波 Superchain Interop 目標係 OP Mainnet ↔ Unichain，而且先只係原生 ETH。未正式激活並驗證之前，唔好把從 Unichain／OP 過來嘅「即將互通」當成已經入咗 Ink 帳。

鏈 genesis 喺 Superchain registry／ETHBEAT 記錄為 **2024-12-06 15:20 UTC**（香港時間 23:20）；Kraken 公開宣布主網係 **2024-12-18**。對面試同對帳，用 chainId 57073 同合約地址即可，唔使糾結「6 號定 18 號先算上線」。

## 面試短答

1. **Ink 係 Kraken 推出嘅獨立 OP Stack／Superchain L2，主網 chainId 57073，測試網 763373，gas 係 ETH，DA 走以太坊 blob。** 唔係 Ethereum，亦唔係 OP Mainnet（10）或 Base（8453）。公開主網 2024-12-18；ETHBEAT 寫 2025-01-22 起為 Stage 1。
2. **密封塊 1 秒；subblocks（前稱 Flashblocks）約 200ms，只係可被撤回嘅預確認。** 流裏面 `block_hash`／`state_root` 永遠係 0。大額入帳等密封塊再跟 L1 最終性，唔好當已經同 Base Denim 一樣改咗塊時間。
3. **官方暫時冇自建 bridge portal，但 L1 仍有標準 OP Stack 橋。** 經典出金要等挑戰期 + execution delay（ETHBEAT：各 3 日 12 小時，約 7 日）再 L1 finalize。合作橋／CCTP／OFT 要分開對帳，源鏈成功 ≠ 目標鏈到帳。
4. **原生 USDC、USDC.e、USDT0 係三個資產。** Circle 原生 USDC 係 `0x2D27…EAEd`，CCTP domain 21；官方列表上嘅 USDC.e 係 `0xF181…5D14`。唔好複用其他鏈地址，測試網要分開。
5. **Kraken 支援 Ink ≠ 你哋已經接入。** 充值地址、確認政策、資產鍵都要自己建；亦唔好把 Superchain 當成統一餘額。

## 常見追問（連答案）

**Q1：Subblocks 已經見到成功，可唔可以入可用？**  
A：唔可以當最終性。官方寫預確認帶 sequencer 信任假設，unsafe 重組時可以撤回；payload 入面嘅 `block_hash`／`state_root` 永遠係 0。零售小額若產品批准，最多當「已預確認、未結算」。大額等 1 秒密封塊，再按 L1 派生／最終性入帳。游標要記密封塊嘅 tx hash／block hash，唔好追 subblock 嘅零 hash。

**Q2：點解唔可以當 Base Denim 咁，用 200ms 塊掃鏈？**  
A：Base Denim 把密封塊時間本身改到約 200ms。Ink 官方密封塊仍係 1 秒，200ms 係額外嘅 subblocks 流。掃鏈、對帳、確認數要以密封塊為準；subblocks 只適合加速偵測，之後必須用密封塊回補同去重。

**Q3：用戶話已經從以太坊過嚗，但 Ink 帳戶係 0；或者從 Ink 提回主網，L2 成功就要求入現貨帳。點處理？**  
A：先分路線。經典 OP 橋：L1 存款成功可能仲未喺 L2 mint／完成；L2 提款發起後仲有約 7 日（3d12h + 3d12h）先能喺 L1 finalize。合作快橋、CCTP、LayerZero 各有自己嘅「源成功／目標實收」語義。一律只認目標鏈實收，源鏈成功只記在途。

**Q4：見到 ticker 係 USDC 或者同 OP／Base 一樣嘅地址，可唔可以開充提？**  
A：唔可以假設同址。原生 USDC 用 Circle 地址 `0x2D27…EAEd`；USDC.e 係另一個合約 `0xF181…5D14`，唔能當同一個帳本科目。USDT0 用官方 `0x0200…0c1`。測試網 USDC `0xFaba…0Ac` 唔好寫進主網庫。資產鍵 `(57073, contract)`。

**Q5：CCTP domain 21 burn 成功，係咪可以入帳？**  
A：唔可以。同其他 CCTP 鏈一樣，要等目標鏈 mint／Transfer 實收。TokenMessenger 地址同好多鏈相同，但 domain 必須係 21；搞錯 domain 會對錯鏈。

**Q6：同屬 Superchain，可唔可以共用 OP Mainnet 嘅掃鏈、合約同 ETH 餘額？Kraken 都已開 Ink，我哋係咪只差開個網絡選項？**  
A：唔可以。chainId、橋合約、token 地址、確認政策都要獨立接入。Kraken 開咗只代表佢哋自己嘅託管同產品；你哋嘅充值地址必須係自己控制嘅熱錢包或賦址。Superchain interop 首波並非「所有 OP Chain 統一餘額」，Ink 嘅 `lagoon_time` 亦唔等於當日已可把別條鏈嘅 ETH 當 Ink 入帳。

## 小練習（題目 + 參考答案）

**題目：** 組員打算開 Ink 充提，方案係：(1) 複用 OP Mainnet（10）同 Base 嘅 USDC／WETH 地址，確認用「subblocks 成功 + 12 個 L2 確認」；(2) 把官方列表嘅 USDC.e 同 Circle 原生 USDC 合成一個 USDC 科目；(3) 用戶經經典橋從 Ink 提回以太坊，L2 成功即入主網現貨；(4) 認為 Superchain／lagoon_time 已等於同 OP、Unichain 統一 ETH 餘額；(5) 因為 Kraken 已支援，就複用公開文檔入面嘅範例地址做充值。有咩問題？點改？

**參考答案：**

- **獨立鏈、獨立資產鍵。** chainId 57073（測試網 763373）。WETH 用 `0x4200…0006`；原生 USDC 用 `0x2D27…EAEd`，USDC.e 用 `0xF181…5D14`，USDT0 用 `0x0200…0c1`。三者分科目。唔複用 OP／Base 地址。
- **Subblocks 唔係密封塊，更加唔係最終性。** 大額等 1 秒密封塊再跟 L1 最終性；零 `block_hash` 唔能做游標。「12 確認」係主網啟發式，唔適用。
- **經典出金要等約 7 日 L1 finalize。** L2 成功只係發起。CCTP domain 21、快橋、OFT 分開流水，目標鏈實收先入帳。
- **Superchain ≠ 統一餘額。** 未驗證官方互通範圍之前，跨鏈入帳只認本鏈實收。
- **對手所已上線 ≠ 可以借用地址。** 充值地址必須係你哋自己嘅託管地址；公共 RPC 可作備用，生產要有冗餘供應商或自建節點。
