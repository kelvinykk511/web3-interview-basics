# Solana Alpenglow 最終性：CEX 充值入帳同掃鏈（Votor）

> 銜接：[Solana 充值 commitment／reorg](./solana-deposit-commitment-reorg.md) 寫嘅係 TowerBFT 時期「processed / confirmed / finalized」分層。**2026-10-21** Alpenglow（第一期 **Votor**）已上 mainnet：`confirmed` 同 `finalized` 收斂成同一狀態。本文講交易所 scanner／入帳狀態機要改咩。Rotor（取代 Turbine 傳塊）尚未上線，唔好當成已換傳播層。

## 人話（從你熟的 CEX）

以前喺 Solana 入帳，好似信用卡「已授權」同「已結算」分兩檔：小額用快啲但協議上仍可能丟嘅 `confirmed`，大額同日結先等要十秒級嘅 `finalized`。Alpenglow 之後，驗證者唔再把投票當成區塊入面嘅交易，而係互相直接投票、聚合成證書；一個 block 一兩輪就可以最終確定。官方目標大約 **150ms**（期望值，唔係常數；視乎權益分佈同一輪定兩輪）。

實際效果好簡單：**而家用 `finalized` 入可用餘額，幾乎同以前用 `confirmed` 一樣快，但保證更強。** 新碼應該直接綁 `finalized`；`confirmed` 暫時仲可以用，但官方話之後會 deprecate 再移除。`processed` 冇變——只代表某 leader 嘅某個 bank 執行過，slot 可以有多個候選塊，呢個 bank 可能被棄。**仍然唔可以喺 processed 就加可用餘額。**

掃鏈／Geyser 要注意：同一個 slot 可以有多於一個候選塊（各係一個 bank）。Agave 4.3 開始，Geyser／Yellowstone 事件帶 `bank_id`。緩衝鍵係 `(slot, bank_id)`，只保留去到 `Confirmed` 嗰個 bank，其餘丟掉。`bank_id` 係**單節點本地計數**，唔可以攞嚟跨 RPC／跨連線對齊；跨流要對 **blockhash**。

投票交易已離開區塊。TPS／交易數儀表若把舊 baseline（裡面大量 vote tx）直接比較，會以為網絡「塌咗」——要 rebase，唔好當成真實吞吐回退，更唔好誤觸熔斷。出金仍應等 `finalized`（而家好平）；偵測到帳可以繼續用 signature 訂閱，唔好死輪詢等一個「確認數」。

主網時間（方便對帳）：epoch 1064 於 **2026-10-21 21:36（香港時間，13:36 UTC）** 開始；特徵閘開啟後仍用 TowerBFT 走約 5,000 個 slot（約 18 分鐘），其後喺 **slot 459,653,000** 切到 Alpenglow。區塊時間減到 ~200ms 係另一項、已於 2026 年 10 月完成嘅改動，同「最終性 150ms」唔係同一件事。

## 面試短答

Alpenglow 第一期 Votor 已取代 TowerBFT：最終性由十秒級變成約 150ms 證書。`confirmed` 同 `finalized` 而家指同一個已出最終確定證書嘅狀態，CEX 新入帳路徑應直接用 `finalized`（細額同大額都可以），`processed` 只做 UI「見到」，因為候選 bank 仍可被棄。Geyser 按 `(slot, bank_id)` 緩衝、以 blockhash 跨節點對齊；唔好用 `getSignatureStatuses.confirmations >= N` 當確認數（未最終確定可能一直係 0，最終確定後係 `null`），改睇 `confirmationStatus`。投票交易唔再在塊裡，監控基線要重設。

## 常見追問（連答案）

**Q1：咁係咪可以 processed 都入帳？終究 150ms 就最終確定？**  
A：唔可以。150ms 係最終確定嘅目標延遲，`processed` 仍然可能喺 leader 棄 bank／`UpdateParent` 之後消失。用戶若已用「可用餘額」買幣或提現，冲正就係事故。正確係：processed 只標 `seen`；加可用餘額等 `finalized`。因為 `finalized` 而家同舊 `confirmed` 一樣快，再為咗快而降級冇好處。

**Q2：舊政策「小於 1 萬等 confirmed、大額等 finalized」仲要唔要？**  
A：分層嘅延遲理由已經冇咗——兩檔而家係同一狀態。可以統一改成全部 `finalized` 先 credited。舊碼繼續讀 `confirmed` 暫時唔會錯（語義已等同），但新碼唔好再寫死 `confirmed`，以免官方移除呢個級之後先爆。

**Q3：`getSignatureStatuses` 嘅 `confirmations` 可以點用？**  
A：唔好當成 EVM 嗰種「再等 N 個 block」。呢個欄位源自舊 TowerBFT lockout；Alpenglow 下，未最終確定時可能一直係 `0`，最終確定後係 `null`（`null` 代表已 rooted／已最終確定，唔係「零確認」）。入帳同出金確認一律睇 `confirmationStatus`（或直接用 `finalized` commitment 去 `getTransaction`）。訂閱（`signatureSubscribe`）通常靚過密密 poll。

**Q4：多個 Geyser／RPC 供應商點合併？TPS 跌咗係咪故障？**  
A：單連線內用 `(slot, bank_id)`；跨連線 **唔好**用 `bank_id`（A 節點嘅 7 同 B 節點嘅 7 唔係同一個 bank）。封塊之後用 **blockhash** 對齊。投票交易離開區塊會令交易筆數大跌（舊時 vote 可佔過半），要 rebase 儀表同告警；呢個唔係吞吐事故。`Clock.unix_timestamp` 改由該 slot leader 喺協定上下限內填寫，唔好再當 stake 加權中位數去對秒級結算。

## 小練習（題目 + 參考答案）

**題目：** 舊政策：小於 1 萬 USDC 喺 `confirmed` 入可用，大額等 `finalized`。Alpenglow 上線後，scanner 用 Geyser `processed` 見到 signature S 入用戶 ATA，200ms 內就 credited（想「反正好快會最終確定」）。同時有 job 用 `confirmations >= 32` 先把出金標成功；上線後呢批出金單一直停喺廣播成功、確認中。TPS 告警亦因為「交易數跌六成」觸發，值班暫停咗 SOL 充提。有咩問題？點改？

**參考答案：**

- **入帳**：唔可以因為最終性變快就改用 `processed`。S 所在 bank 仍可能被棄。改為 `seen@processed`，**全部額度都等 `finalized` 先加可用**（而家延遲同舊 `confirmed` 相近）。
- **出金**：停用 `confirmations >= 32`。改睇 `confirmationStatus == finalized`（或 `getTransaction` 用 `finalized`）。`confirmations` 喺最終確定前可以一直係 0，確定後係 `null`，等 32 會永久卡住。
- **告警**：交易數下跌係投票交易離塊，唔係網絡停擺。rebase baseline；真停充提要靠最終性延遲、RPC 錯誤率、reorg／dead slot，而唔係原始 tx count。
- **掃鏈**：Geyser 緩衝 `(slot, bank_id)`，只 promote 去到 Confirmed／finalized 嘅 bank；跨節點用 blockhash 對齊。
