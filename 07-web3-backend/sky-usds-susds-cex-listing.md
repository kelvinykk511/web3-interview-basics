# Sky USDS／sUSDS：CEX 上架、入帳同庫存（同 DAI／USDe 分開）

> 銜接：ERC-4626 份額記帳見 [erc4626-vault-shares-cex.md](./erc4626-vault-shares-cex.md)；昨日 Ethena USDe／sUSDe（合成美元、鑄贖白名單、解質押有 cooldown）見 [ethena-usde-susde-cex-listing.md](./ethena-usde-susde-cex-listing.md)；穩定幣凍結見 [stablecoin-blacklist-freeze.md](./stablecoin-blacklist-freeze.md)；跨鏈官方橋 vs 自建調撥見 [layerzero-oft-vs-cctp-rebalancing.md](./layerzero-oft-vs-cctp-rebalancing.md)、[usdt0-omnichain-listing-deposit.md](./usdt0-omnichain-listing-deposit.md)；L2 提回 L1 延遲見 [l2-deposit-finality.md](./l2-deposit-finality.md)。

官方說明：[USDS](https://developers.skyeco.com/protocol/tokens/usds/)、[sUSDS](https://developers.skyeco.com/protocol/tokens/susds/)、[Protocol Token Routes](https://developers.skyeco.com/quick-start/protocol-token-routes/)、[LitePSM](https://developers.skyeco.com/protocol/liquidity/litepsm/)、[Spark sUSDS Token](https://docs.spark.finance/dev/savings/susds-token)、[SkyLink Ethereum–Base](https://developers.skyeco.com/guides/skylink/base-eth-native-bridge/)、[USDS Ethereum–Solana Bridge（遷移公告）](https://developers.skyeco.com/guides/skylink/usds-ethereum-solana-bridge/)。匯率同橋狀態會變，以鏈上同官方公告為準。

## 人話（從你熟的 CEX）

USDS 係 Sky Protocol（前 MakerDAO）嘅美元穩定幣，官方寫係 **DAI 嘅升級版**，並同 DAI **共用同一發行來源**，用 **DAI-USDS Converter** 做 **固定 1:1**、**唔收費用**、而且呢條路線 **以後亦不能開費**；官方仲寫呢條轉換 **冇流動性限制**。呢點同 USDC／USDT 嘅發行方鑄贖、亦同昨日講嘅 USDe（許可制 mint／redeem、可拒單）都唔同。

但 1:1 **唔等於同一個 asset_id**。USDS 同 DAI 係兩隻 ERC-20、兩個合約；客戶充 USDS 就要入 USDS 帳，禁止因為「隨時可兑」就自動記成 DAI 或可用 USDC。USDS ↔ USDC 亦**唔行**呢條 Converter，而係走 Ethereum 上嘅 **LitePSMWrapper-USDS-USDC**（官方：而家未開費，**費用以後可以改**）。LitePSM 用 **預鑄好嘅 DAI／USDS 池** 做兩次 ERC-20 轉帳、官方寫無滑點——池有限，唔係無限對手方。

sUSDS 係把 USDS 存入 Sky Savings Rate（SSR）模組之後拿到嘅 **ERC-4626 份額**（枚數唔 rebase，匯率向上；對比 stETH）。官方：呢條 USDS ↔ sUSDS 路線 **唔收費用、以後亦不能開費**；Spark 整合文件寫可以即時、無滑點、無流動性約束噉兑——呢點同 sUSDe 要 request → cooldown → withdraw **剛剛相反**。SSR 係治理設定、用協議盈餘資助嘅可變年率，**唔保證、唔好寫死 APY**。sUSDS 同 sDAI **唔能 1:1 互换**（官方：面值同已累積收益都唔同）。

合約本身（官方＋現行實作）係 **UUPS／ERC-1967 可升級**；USDS 仲支援 permit 同 EIP-1271。現行 `Usds.sol` 嘅 `transfer` 只檢查地址同餘額／授權，**冇** USDC 式 blacklist——但 `wards` 可以升級實作同 `mint`，所以「而家唔能凍結」唔能當成永遠不變。要監控實現地址同授權變更。

跨鏈亦唔能當成「同一個 USDS 口袋」。官方 SkyLink：Ethereum ↔ Base 用改過嘅 OP Stack 原生橋（L1 lock／L2 mint；L2 提回 L1 有大約 **7 日** 安全期）。Ethereum ↔ Solana 官方橋喺 **2026-11-17 14:00 UTC（香港時間 11-17 22:00）** 開始，由 Wormhole NTT 遷移去 LayerZero OFT Adapter，宣布停機大約 31 小時，預計至 **2026-11-18 21:00 UTC（香港時間 11-19 05:00）**，若檢查提早最早約 **17:00 UTC（11-19 01:00）**。公告寫：token 地址不變、鏈上結餘照常用；但跨鏈會暫停，橋合約同呼叫入口會換；未完全激活前唔好經新 LayerZero 橋發起（到不了目標鏈）。本篇寫作時（2026-11-18 早上，香港時間）仍喺宣布窗口內——金庫調撥要對官方／治理執行結果，唔好假設已經好返。

## 面試短答

1. **DAI ≠ USDS ≠ sUSDS ≠ sDAI ≠ USDC ≠ USDe。** 入帳按 `(chain, contract)`。DAI↔USDS 官方 1:1、免稅、無流動性限制，只代表**經 Converter 做過**先可以改資產；唔係掃到充值就合併帳本。
2. **USDS↔USDC 有獨立容量同費率風險。** 走 LitePSM，用預鑄池；而家未開費，但官方寫費用可以改。庫存「隨時兑 USDC」要計池深度同暫停／參變，唔好抄 Converter 嘅「不能開費、無限額」假設。
3. **sUSDS 用份額記帳，唔好 1:1 當 USDS。** 入帳同負債記 sUSDS shares，展示用 `convertToAssets`（Spark 寫即使 `drip()` 未叫都會投影 `chi`）。協議匯率 ≠ 二級市場價；標記價同 haircut 分開。Earn 若自動包裝，出金 USDS 要經 `redeem`／`withdraw`，並檢查 `maxRedeem`，雖然官方路線即時免稅，仍然唔能同「帳本已係可用 USDS」混為一談。
4. **同 sUSDe 嘅對比係常規追問。** sUSDS：即時兑回 USDS、無 cooldown（官方／Spark）；SSR 可變可被治理調低。sUSDe：有 Silo 冷卻，鑄贖仲要白名單。兩者都不能當成 USDC。
5. **可升級＋跨鏈橋 ≠ 地址不變就一切照舊。** 監控 proxy implementation／wards。Base 提回 L1 約 7 日。Solana 官方橋遷移期間：token 地址不變唔代表橋仲係舊合約；停機時唔好把在途 ETH↔SOL 調撥當已到帳，亦唔好喺未宣布完成前改打新 OFT 入口。

官方 Ethereum 地址（Spark／SkyLink）：USDS `0xdC035D45d973E3EC169d2276DDab16f1e407384F`，sUSDS `0xa3931d71877C0E7a3148CB7Eb4463524FEc27fbD`。Base：USDS `0x820C137fa70C8691f0e44Dc420a5e53c168921Dc`，sUSDS `0x5875eEE11Cf8398102FdAd704C9E96607675467a`。Solana USDS mint：`USDSwr9ApdHk5bvJKMjzff41FfuX8bSxdKcR81vTwcA`。上架以官方頁為準，唔好抄錯鏈。

## 常見追問（連答案）

**Q1：DAI 同 USDS 官方 1:1，點解唔合併成一個資產？**  
A：合約、許可、充值掃描、用戶意圖都唔同。1:1 係 **Converter 路徑** 嘅性質，唔係「收到邊隻都當同一隻」。自動幫客兑來兑去要有明確產品授權。保證金同出金要出用家指定（或產品條款寫明）嘅那隻幣。

**Q2：sUSDS 既然可以即時兑 USDS，Earn 可唔可以當可用 USDS 墊付出金？**  
A：技術上兑回路徑同 sUSDe 冷卻唔同，風險細啲，但仍然係兩本帳：客戶負債若記 sUSDS shares，你墊付 USDS 就係用自有庫存借出，要計匯率變動、升級風險同 `maxRedeem`。正確係 redeem 成功（或明確庫存限額）先加可用 USDS，而唔係充到 sUSDS 就 1:1 入 USDS 餘額。

**Q3：Solana 上嘅 USDS 同 Ethereum 係咪永遠 1:1，地址又冇改，橋升級係咪可以當無事？**  
A：官方寫遷移後 Solana USDS 仍然由 Ethereum 上嘅 USDS **1:1 支持**，**token 地址不變**；但 **橋合約同呼叫入口會換**（Wormhole NTT → LayerZero OFT Adapter），停機期間新跨鏈會暫停，未完全激活就發起嘅新 OFT 單到不了目標鏈。CEX 要用 `(chain, contract)` 分開庫存，調撥只認官方路由且目標鏈實收；地址不變 ≠ 繼續打舊 NTT Manager。

## 小練習（題目 + 參考答案）

**題目：** 產品把 USDS 標成「就係 DAI／同 USDC 一樣」。後台把用戶充入嘅 sUSDS 1:1 記入可用 USDS；又把 Ethereum、Base、Solana 嘅 USDS 合成一個全球庫存。Solana 橋宣布遷移停機期間，財務見到 Ethereum 熱錢包已 lock／授權，就先加 Solana 可用額，並改打新 LayerZero 入口想「趕在停機完之前到帳」。有咩問題？點改？

**參考答案：**

- **命名同風控：** USDS 係 DAI 升級線上的獨立資產，唔係 USDC，亦唔係 USDe。分開 asset_id；DAI↔USDS 要經 Converter 先轉檔。
- **sUSDS ≠ USDS：** 入帳記 shares，展示用 `convertToAssets`；禁止 1:1 撥入可用 USDS。Earn 出金先 redeem，或用單獨限額嘅自有 USDS，並披露 SSR 可變。
- **USDC 出口唔能假設無限 1:1：** LitePSM 有預鑄池，費用可改。大額平倉要睇池同備用二級流動性。
- **跨鏈唔能合成一個口袋：** Base 提回 L1 約 7 日；Solana 要走官方橋。停機期間唔好預入目標鏈；未宣布完成前唔好把新 OFT 當已開通。認目標鏈實收，舊在途單跟舊橋 finalization（公告寫停機前已發起嘅 Wormhole 單會繼續完成、唔取消）。
- **升級風險：** 監察 UUPS implementation／wards，唔好把「現行 transfer 無黑名單」寫成永久保證。
