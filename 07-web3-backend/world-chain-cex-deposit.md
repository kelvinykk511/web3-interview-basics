# World Chain：對 CEX 上架／充提

> 銜接：你已經識 [EVM 兼容鏈接入差異](./evm-compatible-chain-integration-differences.md)、[L2 充值最終性](./l2-deposit-finality.md)、[OP Superchain Interop](./op-superchain-interop-cex.md)、[Ink（另一條獨立 OP Stack L2）](./ink-op-stack-cex-deposit.md)、[USDT0](./usdt0-omnichain-listing-deposit.md)、同 [USDC CCTP](./usdc-cctp-cross-chain-rebalancing.md)。World Chain 係 World（前 Worldcoin）推出嘅 **OP Stack／Superchain L2**，定位偏「真人」（World ID）優先區塊空間——對其他交易所嚟講，佢只係**另一條獨立鏈**，唔係 Ethereum、唔係 OP Mainnet，亦唔係「有 World App 就可以共用充值地址」。

官方（以 2026-11-27 文件為準）：[Network Information](https://docs.world.org/world-chain/quick-start/info)、[Useful Contract Deployments](https://docs.world.org/world-chain/reference/useful-contracts)、[Funding a Wallet／原生橋](https://docs.world.org/world-chain/quick-start/fund-wallet)、[公開主網宣布 2024-10-17](https://world.org/blog/announcements/world-chain-now-open-every-human)、[Superchain registry `worldchain.toml`](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/worldchain.toml)。原生 USDC／CCTP V2：[Circle 2025-06-11](https://www.circle.com/blog/now-live-native-usdc-and-cctp-v2-on-world-chain)、[USDC 地址](https://developers.circle.com/stablecoins/usdc-contract-addresses)、[CCTP domains](https://developers.circle.com/cctp/references/contract-addresses)。WLD 合約：[World Whitepaper Tokenomics](https://whitepaper.world.org/designing-for-scale/2025-04-28)（Ethereum `0x163f…8753`，World Chain `0x2cFc…3003`）。階段同出金窗口：[ETHBEAT World Chain](https://ethbeat.org/layer2s/projects/world)（寫作日評估：尚未達 Stage 0，主要因為許可制欺詐證明）。

## 人話（從你熟的 CEX）

World Chain 係一條 **EVM 兼容、OP Stack、Superchain** 嘅以太坊 L2。Gas／原生貨幣係 **ETH**（唔係 WLD），數據可用性走 **以太坊 blob**（registry：`data_availability_type = "eth-da"`）。`governed_by_optimism = false`。對你哋嚟講，World App 或者某對手所已經支援，只代表**佢哋自己嘅產品**接咗呢條鏈，唔代表你可以複用充值地址、合約配置，或者把用戶「從 World App 提到 World Chain」當成已經入咗你哋帳。

網絡參數：

| | Mainnet | Testnet（World Chain Sepolia） |
| --- | --- | --- |
| Chain ID | **480**（`0x1e0`） | **4801**（`0x12C1`） |
| EIP-3770 短名 | `wc` | `wcsep` |
| Gas／Currency | ETH | ETH |
| 出塊時間 | **2 秒**（官方同 registry `block_time = 2` 一致） | 2 秒 |
| 區塊 gas（官方頁，寫會不定期上調） | Limit 80M／Target 40M | Limit 30M／Target 15M |
| 公共 RPC 例子 | `https://worldchain-mainnet.g.alchemy.com/public` | `https://worldchain-sepolia.g.alchemy.com/public` |
| Explorer | worldscan.org | worldchain-sepolia.explorer.alchemy.com |
| 官方橋介面 | worldchain-mainnet.bridge.alchemy.com | worldchain-sepolia.bridge.alchemy.com |

官方 docs 嘅硬分叉表只列到 Holocene（2025-01-30 12:00 UTC）；Superchain registry 仲有之後已排期／已激活嘅時間：Isthmus **2025-11-25 12:00 UTC**（香港時間 20:00）、Jovian **2026-05-01 00:00 UTC**（香港時間 08:00）、Karst **2026-09-21 12:00 UTC**（香港時間 20:00）。上架同節點要以 registry／自己對過嘅節點為準，唔好只抄一份可能過時嘅 docs 表。官方未把密封塊寫成 sub-second，亦**未**喺呢頁定義類近 Ink subblocks／Base Flashblocks 嘅預確認流——唔好未核對就當有 200ms 預確認。

確認層級建議：

- **2 秒 L2 塊（unsafe／latest）：** 已封塊，但仍受 sequencer 同之後 L1 派生影響。零售小額若產品批准，都要清楚風險。
- **安全／最終性：** 要等派生到 L1、再繼承以太坊最終性。大額、出金對沖、不可逆操作跟呢一層，唔好數「12 個 L2 確認」呢套主網啟發式。

經典橋喺 L1 仍然在，registry 地址為：

- L1 Standard Bridge（以太坊主網）：`0x470458C91978D2d929704489Ad730DC3E3001113`
- OptimismPortal（以太坊主網）：`0xd5ec14a83B7d95BE1E2Ac12523e2dEE12Cbeea6C`
- L2 Standard Bridge（OP Stack 預部署，通常為）：`0x4200000000000000000000000000000000000010`

官方 Funding 頁寫，經原生 OP Stack 橋從 World Chain 提回以太坊，要等故障證明挑戰期，大約 **7 日**。ETHBEAT 寫 interactive fraud proof 嘅挑戰期 **3 日 12 小時**，再加 **3 日 12 小時** execution delay，合共約 7 日先能喺 L1 取回。L2 發起成功 ≠ 主網已到帳。Sequencer 故障時，用戶可經 L1 強制送交易，最多有 **12 小時** 延遲。

風險評估要單獨看，唔好因為「都係 OP Stack／Superchain」就抄 Base 或 Ink 嘅結論。ETHBEAT 喺寫作日把 World Chain 列為**尚未達 Stage 0**（意見式成熟度評級，唔等於「唔安全就一定唔上架」）：欺詐證明目前係許可制，允許提出同挑戰嘅實體好少；升級**冇**用戶退出窗口（可即時升級）；許可制 proposer 失效時，經典出金可能凍結。呢啲要寫進鏈風險評估，但唔能單憑評級自動拒絕或接受。

代幣合約係高頻陷阱：

- WETH：`0x4200000000000000000000000000000000000006`（OP Stack 預部署）
- 原生 USDC：`0x79A02482A880bCe3F13E09da970dC34dB4cD24D1`（測試網 `0x66145f38cBAC35Ca6F1Dfb4914dF98F1614aeA88`，唔好寫進主網庫）
- WLD（World Chain）：`0x2cFc85d8E48F8EAB294be644d9E25C3030863003`
- 官方列表仲有 WBTC `0x03c7…cfa3`、sDAI `0x859d…be5d` 等，一律用 `(480, contract)` 做資產鍵

Circle 喺 **2025-06-11** 宣布，World Chain 上原先嘅 Bridged USDC Standard **原地升級**成原生 USDC，**合約地址不變**。呢點同 Ink 唔同：Ink 而家仍然分開原生 USDC 同 USDC.e 兩個合約；World Chain 官方列表而家就係呢一個 USDC 地址，**唔好再另開一個 USDC.e 科目**（除非你另行發現仲有第二個流通合約，並且完成盡調）。跨鏈優先 CCTP V2，domain 係 **14**（唔係 Ink 嘅 21），TokenMessengerV2 地址 `0x28b5a0e9C621a5BadaA536219b3a228C8168cf5d`（多鏈同址，所以 **domain 必須寫對**）。源鏈 burn 成功 ≠ World Chain 已 mint 入帳；入帳只認目標鏈實收。官方仍列 L1 OpUSDCBridgeAdapter `0x153A69e4bb6fEDBbAaF463CB982416316c84B2dB`，上架時要核對呢條舊橋而家仲開唔開，唔好同 CCTP 流水混在一齊。

WLD **唔係** gas，亦唔能複用其他鏈地址。Whitepaper 寫以太坊 WLD 係 `0x163f8C2467924be0ae7B5347228CABF260318753`，OP Mainnet 仲有另一個地址；用戶把主網 WLD 轉去你哋嘅 World Chain 充值地址，係錯鏈，唔能當已入帳。

World ID 優先區塊空間係呢條鏈嘅產品特點，但 CEX 熱錢包通常**冇** World ID，出金**唔好假設**會享有優先包容或補貼 gas。掃描入帳仍然只認你哋控制嘅地址同目標鏈實收，唔好把「已做 World ID 驗證」當成鏈上最終性。

費用仍係 L2 執行費 + 發上以太坊嘅 L1 security／data fee；熱錢包要備 **ETH**，唔係備 WLD。Superchain 成員身分 **唔等於** 統一 ETH 餘額。官方 Superchain Interop 首波目標亦唔係「所有 OP Chain 一齊互通」；未驗證 World Chain 已納入並激活嘅範圍之前，唔好把從 OP／Base／Unichain／Ink 過來嘅「即將互通」當成已經入咗 World Chain 帳。

鏈 genesis 喺 registry 記錄為 L2 time **1719335639**（2024-06-25 17:13:59 UTC，香港時間 2024-06-26 01:13:59），同開發預覽／ETHBEAT 追蹤起始接近；公開宣布主網係 **2024-10-17**。對面試同對帳，用 chainId 480 同合約地址即可，唔使糾結「6 月定 10 月先算上線」。

## 面試短答

1. **World Chain 係 World 推出嘅獨立 OP Stack／Superchain L2，主網 chainId 480，測試網 4801，gas 係 ETH，DA 走以太坊 blob。** 唔係 Ethereum，亦唔係 OP Mainnet（10）或 Base（8453）。公開主網 2024-10-17。密封塊 **2 秒**。
2. **經典出金要等約 7 日 L1 finalize**（ETHBEAT：3d12h 挑戰期 + 3d12h execution delay）。L2 發起成功 ≠ 主網到帳。ETHBEAT 寫作日評估尚未達 Stage 0（許可制證明、即時升級、無退出窗口），要放入風險評估，唔好自動抄 Base／Ink。
3. **原生 USDC 係 `0x79A0…24D1`，2025-06-11 由橋接 USDC 原地升級，地址不變。** CCTP domain **14**；源鏈 burn ≠ 本鏈入帳。唔好再另開 USDC.e 科目，亦唔好複用其他鏈 USDC 地址。
4. **WLD 喺 World Chain 係 `0x2cFc…3003`，唔係 gas，亦唔等於以太坊 `0x163f…8753`。** 資產鍵一律 `(480, contract)`。
5. **World App／對手所已支援 ≠ 你哋已接入；Superchain ≠ 統一餘額。** World ID 優先區塊空間唔等於 CEX 出金有優先，更加唔等於最終性。

## 常見追問（連答案）

**Q1：2 秒塊已經出咗，可唔可以當最終性入可用？要唔要當成 Ink subblocks 或者 Base Denim？**  
A：2 秒係密封 L2 塊，仍不是 L1 最終性。大額等 L1 派生／最終性。「12 確認」係主網啟發式，唔適用。官方網絡頁寫 2 秒出塊，未把密封塊寫成 200ms，亦唔好未核對就套用 Ink 嘅 subblocks 或 Base Denim 邏輯。

**Q2：用戶話已經從以太坊過嚗，但 World Chain 帳戶係 0；或者從 World Chain 提回主網，L2 成功就要求入現貨帳。點處理？**  
A：先分路線。經典 OP 橋：L1 存款成功可能仲未喺 L2 完成；L2 提款發起後仲有約 7 日先能喺 L1 finalize。CCTP、快橋各有自己嘅「源成功／目標實收」語義。一律只認目標鏈實收，源鏈成功只記在途。

**Q3：見到 ticker 係 USDC／WLD，或者同以太坊、OP、Base 好似嘅地址，可唔可以開充提？**  
A：唔可以假設同址。USDC 用 Circle／官方 `0x79A0…24D1`（domain 14）；WLD 用 `0x2cFc…3003`，唔能入以太坊 `0x163f…8753` 嗰個科目。測試網 USDC `0x6614…eA88` 唔好寫進主網庫。資產鍵 `(480, contract)`。

**Q4：CCTP domain 14 burn 成功，係咪可以入帳？搞錯域會點？**  
A：唔可以先入帳。要等目標鏈 mint／Transfer 實收。TokenMessenger 地址同好多鏈相同，domain 必須係 14；寫成 0（Ethereum）或 21（Ink）就會對錯鏈。

**Q5：都係 Superchain，可唔可以共用 OP／Ink 嘅掃鏈、橋合約同 ETH 餘額？熱錢包有冇 World ID 優先？**  
A：唔可以共用。chainId、橋、token、確認政策都要獨立接入。World ID 優先區塊空間係給已驗證用戶；CEX 熱錢包通常沒有，出金按普通交易處理，唔好假設補貼或優先包容。Interop 未驗證激活之前，跨鏈入帳只認本鏈實收。

**Q6：ETHBEAT 話尚未 Stage 0、升級冇退出窗口，係咪代表唔能上架？**  
A：呢個係意見式成熟度評級，要寫進風險評估同額度／確認政策，但唔等於自動否決。重點係你清楚：許可制證明、即時升級、proposer 失效時經典出金可能凍結，同「用戶已在 World App 見到餘額」無關。

## 小練習（題目 + 參考答案）

**題目：** 組員打算開 World Chain 充提，方案係：(1) 複用以太坊／OP Mainnet 嘅 USDC、WLD 地址，確認用「2 秒塊 + 12 個 L2 確認」，並當成同 Base Denim／Ink subblocks 一樣有 200ms 預確認；(2) 因為曾經係橋接 USDC，另外再開一個 USDC.e 科目，同而家呢個地址對沖；(3) 用戶經經典橋從 World Chain 提回以太坊，L2 成功即入主網現貨；(4) 認為 Superchain 已等於同 OP、Base、Ink 統一 ETH 餘額，CCTP 隨便填一個 domain；(5) 因為 World App 已支援，就複用文檔範例地址做充值，並假設熱錢包享有 World ID 優先出塊。有咩問題？點改？

**參考答案：**

- **獨立鏈、獨立資產鍵。** chainId 480（測試網 4801）。WETH 用 `0x4200…0006`；USDC 用 `0x79A0…24D1`（2025-06-11 起就係原生，唔好另開 USDC.e）；WLD 用 `0x2cFc…3003`，唔好用以太坊 `0x163f…8753`。
- **2 秒係密封塊，唔係最終性，更加未核對到官方 200ms 預確認。** 大額跟 L1 最終性；「12 確認」唔適用。唔好套用 Base Denim 或 Ink subblocks。
- **經典出金要等約 7 日 L1 finalize。** L2 成功只係發起。CCTP domain 必須係 14，目標鏈實收先入帳。
- **Superchain ≠ 統一餘額。** 未驗證官方互通範圍之前，跨鏈只認本鏈實收。Messenger 同址唔等於 domain 可亂填。
- **World App 已上線 ≠ 可以借用地址。** 充值地址必須係你哋自己嘅託管地址。World ID 優先唔適用於一般 CEX 熱錢包，亦唔改最終性。生產 RPC 要有冗餘，唔好只靠一條公共 Alchemy URL。
