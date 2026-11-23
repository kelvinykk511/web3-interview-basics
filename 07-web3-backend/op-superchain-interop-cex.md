# OP Superchain Interop（Lagoon）對 CEX 充提／節點

> 銜接：你已經識 [L2 充值最終性](./l2-deposit-finality.md)、[跨鏈橋充值](./cross-chain-bridge-deposit.md)、同呢排講過嘅 Base／MegaETH 出塊差異。OP Stack 而家要加一層：**協議內跨鏈訊息**（Interop）。唔係再揾一組第三方橋驗證者，而係目的鏈節點自己驗證「源鏈真係出過呢個 log」。對 CEX 嚟講，危險唔係「多咗一條便宜跨鏈」，而係 `safe`／`finalized` 嘅含義同節點拓撲都會變；用錯 RPC 或者見到 `latest` 就入帳，會把兩條鏈嘅風險綁埋一齊。

官方（以而家文件為準）：[Interop explainer](https://docs.optimism.io/op-stack/interop/explainer)、[Interop 啟用準備](https://docs.optimism.io/notices/interop-prep)、[Upgrade 21](https://docs.optimism.io/notices/upgrade-21)、[交易所／橋安全指引](https://docs.optimism.io/op-stack/security/exchanges-and-bridges)。截至 **2026-11-23，主網 Interop 仍未啟用**。官方目標：Lagoon hardfork 喺 **2026-12-10** 啟用 OP Sepolia（chainId `11155420`）↔ Unichain Sepolia（`1301`），**2027-01-11** 啟用 OP Mainnet（`10`）↔ Unichain（`130`）；精確時間戳未釘，會寫入 Superchain Registry，主網仍待治理批准。首發範圍係呢兩個 **兩鏈 dependency set**，產品面先支援原生 ETH 轉帳，唔包括通用 ERC-20、亦唔係兩條鏈共用一個統一餘額。

Upgrade 21 係 **L1 合約升級**（`op-contracts/v9.0.0`），**冇 L2 hardfork**。官方目標主網執行日係 **2026-11-16**（文件仍標 provisional，日期可以挪）。佢做咗兩件同你有關、但 **唔等於 Interop 已開** 嘅事：每個鏈嘅 ETH 保管同 pause identifier 改為 `ETHLockbox`；Interop 要用嘅 L1 合約已就位，但 **唔會呼叫 `migrate`、亦唔會開 `INTEROP` 功能**。真正合成一個 interop set，要之後先 `OPContractsManagerV2.migrate`：共用 `ETHLockbox`／爭議工廠／`AnchorStateRegistry`，而且係 **單向**（開咗唔能關）。遷移會令遷移前已證明嘅出金 proof **全部失效**，要用家重新 prove。

## 人話（從你熟的 CEX）

而家你掃 OP Mainnet 充值，慣咗對齊 `safe`（低額）同 `finalized`（大額），同官方交易所指引一致：`unsafe`／`latest` 只係 sequencer 聲明，batcher 可以喺上 L1 時改寫，上咗當就無得靠協議追返。呢條規則 **Interop 之後仍然適用**，而且更嚴。

Interop 一筆跨鏈要兩步：源鏈先出一個 log（initiating message），目的鏈再有一筆 executing message，經 `CrossL2Inbox` 聲明「我引用嗰個 log」。目的鏈節點要能證明嗰個 initiating message 已達到同等安全級別，呢個塊先可以升做 `safe`。`CrossL2Inbox` 自己唔讀第二條鏈；交易要用 EIP-2930 access list 預先聲明引用，節點入塊前先核對。

再嚴一層：啟用之後，`op-supernode` **一次過按同一個 L2 timestamp 驗證成個 dependency set**。所以 OP Mainnet 同 Unichain 嘅 `safe`／`finalized` **一齊行**：其中一條鏈嘅 safe head 停（例如 batcher 停止往 L1 貼 batch，或者你自己嗰邊嘅第二條鏈 execution client 掛咗），**兩條鏈** 嘅 `safe`／`finalized` 都會停，即使嗰啲塊完全冇跨鏈訊息。同時 `latest`／unsafe 仍然會經 P2P 向前行。呢個就係經典陷阱：監控如果只睇 `latest`，會以為鏈好好，但其實入帳水位已經凍咗——或者更差，有人用未校驗嘅 `safe` 標籤入咗帳。

節點拓撲亦要改。啟用時間一到，你如果而家只跑 OP Mainnet，都要為 supernode 配 **Unichain 嘅 execution client**（可以唔對外提供 RPC，但一定要有、而且要用 Engine API，唔能用普通 `eth_*` RPC 充數）。艦隊裡每個 `op-node` 都要以 Light CL 指向你自己營運或明確信任嘅 supernode（`--l2.follow.source`）。官方 **唔提供** 公共 follow-source。一個未改拓撲嘅 `op-node` 啟用後仍會跑，但會 **喺未做跨鏈校驗嘅情況下把塊升做 safe**，呢個 `safe`／`finalized` **唔能信**。第三方／公共 RPC 更加係叫你信營運方。

L1 出金 pause 亦要納入同一套值班：pause 唔會停 L2 出塊，交易同 L1→L2 存款仍然會發生。官方話，如果有人喺 L2 偷咗或鑄多咗資產，官方橋一關，交易所同第三方橋就變成出口。所以 pause 期間要 **停入帳、停出金、停往呢條鏈打款**，恢復係人工決策。查 `OptimismPortal.paused()`（唔好只 call `SuperchainConfig.paused()`，嗰個只反映 global pause）。Upgrade 21 之後，鏈專屬 pause identifier 係 `ETHLockbox`；監控 portal 餘額嘅，改睇 lockbox。Polling 仍然必要：pause 約三個月後可能無 event 就過期。

## 面試短答

1. **Interop 未開。** 12-10 測網、01-11 主網只係目標；時間戳未釘，主網要治理批准。Upgrade 21 ≠ 已互通。
2. **入帳規則冇放寬，只係 `safe` 更重。** 低額最少 `safe`，大額 `finalized`；`latest` 仍然只係 sequencer 聲明。啟用後一個塊要等佢引用嘅 initiating message 都達到同等安全級，先算 safe。
3. **兩條鏈嘅 `safe`／`finalized` 會綁定前進。** 一條滯後，兩條都停；`latest` 仍會行。要為兩條鏈嘅 safe head 設告警。
4. **要自建 supernode，同時跑兩條鏈嘅 EL。** 未接 follow-source 嘅 `op-node`，佢報嘅 `safe` 唔能用嚟入帳。
5. **唔係統一餘額，首發亦唔係任意 ERC-20。** 遷移後舊 withdrawal proof 要重做；pause 時停充提，以 `OptimismPortal.paused()` 為準。

## 常見追問（連答案）

**Q1：Interop 開咗，係咪等於用戶喺 Unichain 入 ETH，OP Mainnet 熱錢包會自動加同一筆餘額？**  
A：唔係。官方明確講：首發唔提供跨鏈統一餘額或流動性池。協議讓目的鏈驗證源鏈訊息，原生 ETH 轉帳係一條受支援路徑，但兩邊仍然係獨立帳本。CEX 必須繼續用 `(chainId, asset)` 記賬；用戶撳錯鏈就要走明確嘅跨鏈路徑，唔好當「同一個 ETH 袋」。

**Q2：可唔可以用公共 RPC 嘅 `safe` 標籤，或者啟用後繼續用而家單一 `op-node`？**  
A：公共 RPC（包括鏈官方）要你信營運方。自己嘅節點如果冇指向有雙鏈 derivation 嘅 supernode，啟用後會把未做跨鏈校驗嘅塊標成 `safe`，呢個標籤對入帳冇意義。要入帳，RPC 必須來自你營運或明確信任、而且已完成拓撲遷移嘅節點。

**Q3：Upgrade 21 做完，出金同 pause 監控要改咩？**  
A：ETH 保管喺 `ETHLockbox`，唔好再把 `OptimismPortal` 餘額當成鏈上託管 ETH。Pause identifier 亦係 lockbox，但檢查仍然用 `OptimismPortal.paused()`。真係 `migrate` 合成 interop set 嗰陣，舊嘅 withdrawal proof 會失效，要重新 prove；在途 L2→L1 出金要有重證明流程，唔好當交易失敗就重播同一組 proof。

## 小練習（題目 + 參考答案）

**題目：** 2027-01-11 目標日一到，組員打算：(1) 當 OP Mainnet 同 Unichain 已經係同一個 ETH 餘額，用戶從任一邊充值都入同一本帳；(2) 繼續用而家只跑 OP Mainnet 嘅 `op-node` 同公共 RPC，見到 `latest` receipt 就入可用；(3) 其中一條鏈 `safe` 停咗都當另一條仲正常，因為嗰批塊冇跨鏈訊息；(4) L1 pause 都當唔關事，因為 L2 仲出塊。有咩問題？點改？

**參考答案：**

- **確認 Registry 時間戳同治理，先當啟用。** 目標日唔等於已開。資產 key 仍然係 `(10, ETH)` 同 `(130, ETH)`，首發亦唔當任意 ERC-20 已互通。
- **入可用對齊自己 supernode 嘅 `safe`／`finalized`。** 大額用 `finalized`。`latest` 喺 safe 停滯時仍會前進，唔能當確認。
- **啟用前完成雙鏈 EL + supernode + Light CL。** 未改拓撲嘅 `safe` 唔能信。
- **兩條鏈嘅 safe head 都要告警。** 一條滯後，兩條入帳都要停。
- **Pause：每條支援鏈、每個 L1 block 用 `latest` 查 `OptimismPortal.paused()`。** Pause 期間停入帳同出金；恢復要人工確認 `paused()==false`，唔好只聽 `Unpaused` event。遷移前清點在途 withdrawal proof。
