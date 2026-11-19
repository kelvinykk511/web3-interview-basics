# Plasma 鏈 USDT0：CEX 上架同充提

> 銜接：USDT0／OFT Mesh／Legacy Mesh、資產 key `(chainId, contract)` 見 [usdt0-omnichain-listing-deposit.md](./usdt0-omnichain-listing-deposit.md)；OFT `eid`／guid／DVN、同 CCTP 分工見 [layerzero-oft-vs-cctp-rebalancing.md](./layerzero-oft-vs-cctp-rebalancing.md)。今日落到 **Plasma 呢條穩定幣付款 L1**：用戶口裡嘅「USDT」好可能係打去呢條新鏈，交易所最容易用錯 chainId、抄 Ethereum 確認數、定係同其他鏈嘅相同合約地址混帳。

官方網絡參數：[Plasma Mainnet Details](https://docs.plasma.org/docs/guides/network-configuration/mainnet-details)——名稱 `Plasma Mainnet Beta`，chainId **9745**，原生 gas **XPL**，出塊約 **1 秒**，共識 PlasmaBFT（Fast HotStuff 變體），EVM 兼容，瀏覽器 [plasmascan.to](https://plasmascan.to/)。共識頁寫快路徑兩次連續 QC 即可最終確定、最終性以秒計。USDT0 地址以 [USDT0 Deployments](https://docs.usdt0.to/technical-documentation/deployments) 為準（上線前再核對官方頁）：Plasma Token `0xB8CE59FC3717ada4C02eaDF9682A9e934F625ebb`，OFT `0x02ca37966753bDdDf11216B73B16C1dE756A7CF9`，LayerZero EID **30383**（**不是** 9745）。

## 人話（從你熟的 CEX）

Plasma 係一條專做穩定幣付款嘅 EVM L1，唔係 Ethereum 嘅 L2，亦唔係「另一個 Tron」。用戶從錢包揀 Plasma 網絡出 USDT，到帳嘅係呢條鏈上嘅 **USDT0**（OFT Mesh 嗰張官方合約），背後仍然係 Ethereum 鎖住嘅經典 USDT 1:1 支撐，但帳本必須當成 `(9745, 該合約)` 一隻獨立可充提資產。唔可以同 Ethereum 經典 USDT、Tron／TON 原生 USDT、或者其他 L2 嘅 USDT0 合併庫存——ticker 都叫 USDT，但提現要喺同一條鏈、同一張合約出得返去。

有一個好易踩嘅坑：官方 Deployments 入面，Plasma 嘅 USDT0 **token 地址**同 HyperEVM（chainId 999）、MegaETH（chainId 4326）係**同一個** `0xB8CE59FC…5ebb`。呢個喺 EVM 多鏈好常見（同一 bytecode／CREATE2）。如果你嘅資產 key 只得合約地址、冇 chainId，三條鏈嘅充值會混成一盤，出金就會打錯鏈。

費用同「零手續費」亦唔好聽宣傳就寫死。Learn 頁提過 Plasma 為 USDT0 提供 zero-fee transfers，但 [Mainnet Details](https://docs.plasma.org/docs/guides/network-configuration/mainnet-details) 列明而家 live 嘅 fee token **只有 XPL**，多幣種 gas 寫「soon」；[Network Fees](https://docs.plasma.org/docs/plasma-chain/network-information/network-fees) 亦話自訂 gas token／協議 paymaster 仍在建設。出金熱錢包要備 XPL，唔好假設任意錢包、任意 `transfer` 都可以零 gas 上鏈。Confidential payments、毫秒級 timestamp 文件寫緊發展中／將會提供，唔好當已生效。

最終性同 Ethereum 唔同：唔係 slot 先提案、再等好幾個 epoch 先 `finalized`。PlasmaBFT 以秒計敲定。CEX 應以 RPC `finalized`（或等價最終性標籤）入可用，**唔好**抄主網「12／32 個 confirmation」或者「等 15 分鐘」，亦唔好把示範程式裡嘅 `confirmations: 2` 當生產政策。出塊快 ≠ 見到 `latest` 就可以入帳。

## 面試短答

1. **資產 key 一定係 `(chainId, contract)`。** Plasma 係 9745 + 官方 USDT0 token。只認 ticker「USDT」、或者只認合約地址，會同 HyperEVM／MegaETH 嘅相同地址撞庫。EID 30383 只用於 OFT `send`，唔好同 chainId 對調。
2. **入帳只認呢條鏈、呢張官方合約嘅成功 `Transfer`／mint。** 用戶喺 Ethereum／Tron 鎖倉或 burn 之後，要等 Plasma 上官方合約先出額；唔好喺源鏈成功就加 Plasma 可用。打錯合約（舊橋、假 USDT、其他鏈地址）走錯幣找回，唔入 USDT 帳。
3. **最終性用 BFT 嘅 `finalized`，唔抄 Ethereum 確認數。** 約 1 秒出塊、最終性以秒計，但 `latest`／未最終化仍然可能受 view change 影響。大額照等最終化；reorg rewind 仍然要有，只係窗口同 Ethereum 唔同。
4. **Gas 而家仍係 XPL。** 零費 USDT0 若存在，只係特定產品／paymaster 路徑，唔係協議保證每筆 ERC-20 都免費。熱錢包要監控 XPL 餘額；用戶只有 USDT0 冇 XPL，唔等於佢唔可以經官方 paymaster 轉入，scanner 仍要聽 `Transfer`。
5. **金庫調撥走官方 USDT0 OFT。** Plasma 已在 OFT Mesh 內，原則 1:1 burn／mint（仍要處理 dust／報價）。用 guid 對兩端事件，源成功只准補執行 `lzReceive`，唔好再 `send`。唔好經不明第三方橋先入可用。

## 常見追問（連答案）

**Q1：Plasma 上嘅 USDT 同 Ethereum／Tron USDT 可唔可以合成一個「USDT 總庫存」？**  
A：帳本可以有展示用嘅合計，但可出金庫存必須按 `(chainId, contract)` 分池。Plasma 上係 USDT0，要出返 Ethereum 經典 USDT 要走官方 OFT redeem／Adapter unlock；出 Tron 原生 USDT 更可能行 Legacy Mesh，有 fee 同池子限額。合成一個池會令某條鏈爆倉、另一條鏈出唔到。

**Q2：地址都係 `0xB8CE59FC…`，可唔可以共用同一套充值地址同掃鏈游標？**  
A：唔可以。呢個只係多鏈同一個 token 地址，私鑰／ deriv 路徑可以重用，但掃描游標、確認水位、入帳唯一鍵必須帶 chainId。HyperEVM 或 MegaETH 嘅 `Transfer` 唔能記入 Plasma USDT0。RPC、explorer、reorg 處理亦係分鏈嘅。

**Q3：官方話最終性只要幾秒，係咪 receipt 一出就可以入可用？用戶又話零手續費，熱錢包係咪唔使備 XPL？**  
A：Receipt 喺 `latest` 唔等於最終化。正確係跟 `finalized`（或你們對 BFT 鏈既定嘅最終性標籤），再做冪等 `chainId + txHash + logIndex`。零費唔係 Mainnet Details 上嘅現況保證；廣播出金而家仍要 XPL 付 gas。Paymaster 若上線，先核對白名單同實際扣費再改運維假設。

## 小練習（題目 + 參考答案）

**題目：** 產品要開「USDT-Plasma」充提。資產表用合約 `0xB8CE59FC…5ebb` 做唯一鍵（冇 chainId），掃描器抄 Ethereum 政策「32 個 confirmation」，約要等 32 秒。金庫見到用戶喺 Ethereum 把 USDT 鎖進 OFT Adapter，即時把等額加入 Plasma 可用庫存。出金 bot 假設 USDT0 轉帳零 gas，熱錢包唔備 XPL。另外有一筆 HyperEVM 上同一地址嘅 `Transfer` 被當成 Plasma 充值。有咩問題？點改？

**參考答案：**

- **資產 key 必須加 chainId：** Plasma 係 9745，HyperEVM 係 999，合約可以相同。無 chainId 會把三條鏈混帳，出金打錯鏈。唯一鍵同游標都帶 chainId；只 allowlist 官方 Deployments 當頁地址。
- **唔好抄 32 個 Ethereum confirmation：** 32 秒唔等於 Plasma 最終性定義，亦可能過長或錯單位（應跟 `finalized`，而唔係數塊數當時間）。`latest` receipt 唔入可用。
- **禁止源鏈 lock 預入：** OFT 源成功 ≠ 目標鏈已 mint。只認 Plasma 官方 USDT0 嘅 `Transfer`／`OFTReceived`，用 guid 對帳；失敗只補執行，唔再 send。
- **熱錢包備 XPL：** 而家 live fee token 只有 XPL。零費係產品／paymaster 假設，未核對前按要付 XPL 設計；XPL 不足要告警，唔好讓出金卡死。
- **回歸：** 用官方 Plasma USDT0 打測試充值，必須入 `USDT-Plasma`；用 HyperEVM 同一地址、或 Ethereum 經典 USDT，必須拒收或入另一科目。
