# LayerZero OFT 跨鏈調撥 vs CCTP

> 銜接：原生 USDC 應用 Circle **CCTP** burn→attestation→mint，唔好同 USDC.e 混帳，見 [usdc-cctp-cross-chain-rebalancing.md](./usdc-cctp-cross-chain-rebalancing.md)。今日對焦：**發行方用 LayerZero OFT 嘅資產（同 USDT0）點樣跨鏈**，以及同 CCTP 喺擔保模型、狀態機、重試上有咩唔同。USDT0 產品細節見 [usdt0-omnichain-listing-deposit.md](./usdt0-omnichain-listing-deposit.md)。

## 人話（從你熟的 CEX）

跨鏈調撥熱錢包，本質係「一邊少貨、另一邊多貨」，中間有一段在途。你已經識兩種做法：兩邊都囤庫存自己對沖，或者走官方 **burn/mint**（冇 AMM 池、唔製造 wrapped 分叉）。

**CCTP** 係 Circle 專為 USDC（同 EURC 等官方路徑）而設嘅燒印：信嘅係 Circle attestation。**OFT（Omnichain Fungible Token）** 係 LayerZero 嘅代幣標準：源鏈 **debit**（burn 或 lock）→ 跨鏈訊息 → 目標鏈 **credit**（mint 或 unlock），全球供應對應 1:1。好多新幣同 USDT0 都用呢條路。面試要講清楚：兩者都係「燒咗再印」，但 **擔保方完全唔同**——OFT 信嘅係嗰隻幣自己配嘅 **DVN 安全棧**，唔係 Circle。

## 面試短答

1. **兩種部署**：**Direct OFT**＝代幣合約自己內建跨鏈邏輯；**OFT Adapter**＝既有 ERC-20 唔改，另外一個 adapter **鎖倉或代為 mint/burn**。Adapter 鎖住嘅係托管資產，合約／admin 風險要當上架項嚟審。Fee-on-transfer、rebasing 預設唔支援（`_debit`／`_credit` 假設轉帳無損）。
2. **路徑識別用 eid，唔係 evm chainId**。例如 Ethereum `30101`、BSC `30102`、Arbitrum `30110`、Base `30184`（以 [LayerZero Endpoint 表](https://docs.layerzero.network/v2/developers/evm/oft/quickstart) 為準）。收款人 `to` 同 peer 都係 **bytes32**（EVM 地址左補零）。雙向都要 `setPeer`；peer 配錯，輕則訊息被拒，重則信咗唔對嘅對手合約。
3. **金額同費用**：`SendParam` 帶 `amountLD`、`minAmountLD`。`sharedDecimals` 默認 **6**；18 decimals 嘅幣會先 floor 掉轉換剩餘 **dust** 退回發送方，再送到對端。收據分 `amountSentLD`／`amountReceivedLD`，帳本唔好假設兩者永遠相等（代幣自身 fee、dust）。先 `quoteOFT` 再 `quoteSend` 攞 `nativeFee`／`lzTokenFee`；fee 會變，quote 同 send 要盡量同一刻，唔夠會 revert。
4. **安全棧**：源鏈 MessageLib 請 DVN 驗證、Executor 到目標鏈執行。Required DVN **全部**要過，Optional 達到門檻先可以 commit。**DVN 先係安全邊界；Executor 只係送件**——執行失敗或 Executor 宕機，訊息已 verified 嘅話，**任何人**可以用同一個 guid 重試 `Endpoint.lzReceive`，唔使再從源鏈 send。配置（DVN、確認數、庫）由 token 嘅 Owner／Delegate 控制，上架要審呢個集合同 admin 權限；單 DVN 或可隨便改棧，等於發行方可印。
5. **狀態機同冪等**：`SEND_SENT → SRC_FINAL → DVN_VERIFIED → DST_EXECUTED（lzReceive）→ DST_FINAL`。源鏈 `OFTSent`／receipt 成功 **≠** 目標已入帳。去重用 **guid**（由路徑 + nonce 算出）同路徑 nonce `(srcEid, sender, dstEid, receiver)`。掃描 `OFTSent`／`OFTReceived`，在途要入庫存對帳。
6. **同 CCTP 點揀**：官方 USDC 跨鏈優先 **CCTP**（原生 USDC、Circle attestation），唔好用第三方「USDC OFT／橋包裝」。發行方以 OFT 做官方路徑嘅資產，先走官方 OFT，唔好再經第三方鎖倉橋整出另一隻 wrapped。CCTP 冪等鍵係 `(sourceDomain, nonce)`；OFT 係 guid／路徑 nonce。兩者都係「源鏈成功後禁止再 send，只准補執行目標鏈」。

## 常見追問（連答案）

**Q1：源鏈已經 debit，目標 15 分鐘都未 mint，錢喺邊？可唔可以再 send？**  
A：係**在途資產**（已 burn／lock，等 DVN verify + lzReceive）。用 guid 去 LayerZero Scan 或查目標 Endpoint 係咪已 verified 未 delivered。已 verified 就重試 `lzReceive`（必要時加 gas）；**絕對唔好再 `send` 一次**，否則會第二次 debit，兩筆在途。超時先告警，唔自動重發。

**Q2：DVN 同 Circle attestation 有咩分別？**  
A：CCTP 嘅信任錨係 Circle 對 burn 訊息嘅 attestation，資產範圍基本係官方穩定幣。OFT 嘅信任錨係**該 OApp 配置**嘅 DVN 集合 + 確認數，每隻幣可以唔同，而且 admin 可以改。審查一隻 OFT＝審查呢個安全棧同升級權，唔係「用咗 LayerZero 就等於同 CCTP 一樣安全」。

**Q3：點解要 `minAmountLD`？dust 會唔會令用戶少收？**  
A：`sharedDecimals` 轉換會 floor；如果中間再扣協議 fee，到帳可能低於發送整數。`minAmountLD` 係滑點下限，低過就 revert，避免「以為轉咗 1 個但對端只到一部分」。退回嘅 dust 留源鏈發送方，入帳／調撥要以收據裡嘅 `amountReceivedLD` 為準。

**Q4：調撥要唔要帶 `composeMsg`？**  
A：金庫純調撥通常**唔需要**。compose 會在到帳後再觸發另一筆邏輯（swap、stake），多一層執行失敗同重試（`lzCompose`），攻擊面更大。熱錢包對熱錢包只做純 send 即可。

## 小練習（題目 + 參考答案）

**題目：** 運維要用官方 OFT 把 80 萬枚代幣由 Ethereum 熱錢包調去 Base。Job 見到源鏈 tx 成功、`OFTSent` 已出，15 分鐘後 Base 未見 `OFTReceived`，就用同一金額再 `send` 一次。兩筆源鏈都成功。有咩問題？正確狀態機？

**參考答案：**

- **重複 debit**：第一筆已在途，再 send 等於又 burn／lock 80 萬，現在 160 萬在途，源熱錢包流動性被鎖。錢通常唔會「蒸發」（兩筆都可以到 Base），但庫存同告警會亂，嚴重時觸發低水位。
- **正確做法**：一張調撥單只准一次成功 `send`（狀態 `SEND_SENT` 後禁止再發）。用返回嘅 **guid** 追 DVN／執行狀態。源鏈最終性之後，若已 verified 未 delivered，只重試目標鏈 `lzReceive`；若連 verification 都未過，等確認數／告警 DVN，仍然唔重發。
- **入帳**：目標確認 `OFTReceived`／balance 增加後先把在途轉成 Base 庫存，金額用 `amountReceivedLD`。`quoteSend` 緊貼 send，`minAmountLD` 設合理下限，`to` 用熱錢包地址嘅 bytes32，eid 用 Base `30184` 而唔係 chainId `8453`。
- **回歸**：模擬「源成功、故意延遲 dest」——job 不可發第二筆；模擬 dest 執行 revert——同一 guid 重試成功後先入帳。
