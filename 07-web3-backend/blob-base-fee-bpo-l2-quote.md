# Blob base fee 走勢：BPO 調容量、指數調價、對 L2 出金報價

> 銜接：4844 基本概念（blob vs calldata、唔使自己發 blob tx、同最終性正交）見 [`../03-ethereum/eip4844-blob-fees.md`](../03-ethereum/eip4844-blob-fees.md)；L2 出金成本拆 L2 execution + L1 data／blob 見 [l2-withdrawal-cost-l1-data-fee.md](./l2-withdrawal-cost-l1-data-fee.md)；用戶手續費鎖價見 [withdrawal-fee-pricing-quote-lock.md](./withdrawal-fee-pricing-quote-lock.md)。今日專注 **blob 費點樣指數上升、容量參數點改、報價點唔好寫死**。

## 人話（從你熟的 CEX）

L2（Base、OP、Arbitrum 等）幫用戶出金時，你多數只係喺 L2 發普通轉帳，**唔使自己組 blob**。但 sequencer 要把呢批交易數據貼返 L1，用嘅就係 blob 市場。所以用戶覺得「L2 gas 好平」，你實際成本裡頭往往有一大截係 **L1 blob／data fee**，跟住 blob 市場情緒走，同嗰條 L2 塞唔塞可以完全唔同步。

Blob 費市場似 EIP-1559，但係另一條曲線：每個區塊有 **target**（目標用量）同 **max**（上限）。用得超過 target，`excessBlobGas` 上升，**blob base fee 指數向上**；低過 target 就回落。容量唔係永遠 4844 剛上線嗰陣嘅「每塊 target 3／max 6」：Pectra 加過一次，Fusaka 引入 PeerDAS 之後，仲可以用 **BPO（Blob Parameter Only）** 小分叉單獨調高 target／max，而唔使等下一個大硬分叉。2026 年 1 月 BPO2 之後，mainnet 係 **target 14／max 21**。參數會再變，fee oracle **唔可以寫死舊數字**。

另外，Fusaka 嘅 EIP-7918 幫 blob 費設咗同 execution base fee 掛鈎嘅地板：市場長期空置時，blob fee 唔好再無意義地跌到 1 wei 然後失去價格訊號。所以「blob 好空」都唔等於成本係零。

## 面試短答

Blob base fee 由父塊嘅 `excessBlobGas` 用類似 1559 嘅指數公式算出嚟，獨立於 execution `baseFeePerGas`。用量持續高過當前 target，費會喺幾個區塊內疊乘上升（滿塊會連升；空塊先回落），所以短時間可以貴幾倍，看起來似「突然爆」。工程監控要分開睇：`eth_feeHistory` 嘅 `baseFeePerBlobGas`／`blobGasUsedRatio`，以及 header 嘅 `blobGasUsed`、`excessBlobGas`，唔好只盯 L1 普通 gas。

容量參數（target／max／update fraction）會隨 Cancun → Pectra → Fusaka BPO 改變；BPO 專門用嚟喺大分叉之間加 blob 吞吐量。CEX 含義：(1) L2 出金 fee oracle 要把 **L1 blob 成分**當成獨立訊號；(2) 唔好把「BPO 之後好平」寫成永久固定用戶手續費；(3) 用戶 quote 要有短 TTL，鎖價後差價交易所吃，但要有單筆成本上限同熔斷（blob 費 N 倍於基線就提高報價或延遲非緊急歸集）；(4) 成本帳用 L2 receipt 嘅實際 L1 fee／等效欄位，唔用估算；(5) 硬編碼 3／6 或 6／9 會令容量告警同滿塊判斷全部錯。PeerDAS 解決嘅係節點點樣抽樣證明數據可用，從而先敢加容量；佢本身唔保證費用永遠平——需求跟上 target，指數曲線仍然成立。

## 常見追問（連答案）

**Q1：點解 L2 出金費可以幾粒鐘貴幾倍，但 L1 轉 ETH 估價幾乎冇變、嗰條 L2 亦唔塞？**  
A：你睇錯市場。Execution base fee 同 blob base fee 係兩條 1559。L1 普通轉帳只買 execution gas；L2 交易成本含 sequencer 攤分嘅 blob／data 費。Blob 需求（好多條 rollup 同時滿貼、或者某條鏈突然大量發 blob）可以單獨把 `baseFeePerBlobGas` 推高。排查順序：L2 execution gas → 該鏈 oracle／receipt 嘅 L1 fee 成分 → L1 `baseFeePerBlobGas`。確認係 blob 之後，加 tip 冇用。

**Q2：BPO 加咗容量，出金係咪以後都平？**  
A：唔係。BPO 只提高 target／max，令同樣需求之下唔咁易超標。若需求（各 L2 貼資料量）很快填滿新 target，超標之後費仍然指數升。正確預期：升級當日可能明顯回落，之後回到「跟用量走」。產品唔可以因為「1 月加過容量」就永久鎖定低手續費。

**Q3：Fee oracle 應該讀邊啲欄位？可唔可以假設每塊最多 6 個 blob？**  
A：用 `eth_feeHistory` 拿最近一段嘅 `baseFeePerBlobGas` 同 `blobGasUsedRatio`，必要時再對當前頭塊讀 `excessBlobGas`。滿塊判斷要用 **當前** max（BPO2 之後係 21，唔係 6 或 9）。客戶端 genesis／`blobSchedule` 會按時間分段；監控設定同 runbook 要當成可配置項，跟升級日曆更新。L2 側仍然用該鏈自己嘅 `getL1Fee`／`gasEstimateL1Component`，因為各鏈點把 blob 成本折進用戶 tx 唔一樣。

**Q4：EIP-7918 對 CEX 有乜實務影響？**  
A：當 blob 市場好空，舊機制可以把 blob base fee 壓到接近 1 wei，看起來「數據免費」；但其實 sequencer 仲要付 execution gas 去驗證 blob。7918 設地板，令空市時 blob 費唔會完全失去意義。報價模型如果假設「空置 = blob 成本可忽略」會低估。監控上：blob fee 唔跌到零係正常，唔好當成 oracle 壞咗。

**Q5：用戶手續費應該跟住每塊 blob fee 即時跳動嗎？**  
A：唔應該對用戶每塊改價，但亦唔可以固定幾個星期。做法同出金鎖價一致：後端按「L2 execution + 當前 L1／blob 成分 + buffer」計動態價，生成短 TTL quote（幾十秒到一兩分鐘）綁 userId＋鏈＋幣；確認後鎖價，之後廣播成本波動由交易所 fee P&L 承擔。另外設熔斷：blob 費相對 24h 基線升破倍數，就提高 buffer、暫停非緊急 sweep／再平衡，或者對該鏈出金改排隊等回落（要有 SLA，唔好無限等）。

**Q6：同充值確認數、L2 最終性有關嗎？**  
A：無關。Blob 市場改嘅係 **DA 刊登成本**，唔改 optimistic 挑戰窗或者 ZK 出證明時間。唔可以因為 blob 平咗或者容量加咗就自動減確認數。

## 小練習（題目 + 參考答案）

**題目：** 產品喺 BPO2 後見 Base 出 USDT 好平，把該鏈用戶手續費寫死成「每筆 0.15 USDT」。監控只睇 L1 `baseFeePerGas` 同該 L2 嘅 execution base fee。某日一堆 L2 同時把 blob 用量頂到靠近 max（21），連續好多塊超 target（14），blob base fee 幾小時內疊乘上升。熱錢包 ETH 消耗快過預算，fee P&L 轉負，但兩條「gas 告警」都沒響。問題喺邊？點改？

**參考答案：**

- **漏咗獨立訊號**：成本大頭係 blob／L1 data fee，唔係 L1 轉帳 gas，亦唔一定係 L2 execution。要對 `baseFeePerBlobGas`（以及該鏈 receipt／oracle 嘅 L1 fee）單獨設倍數告警。
- **寫死手續費**：BPO 後嘅便宜期唔代表結構性永久低成本。改成動態報價＋短 TTL 鎖價；固定費若要保留，必須加上下限同「超過上限就切動態／暫停」。
- **容量參數**：滿塊／超標判斷用當前 14／21（或之後下一次 BPO 嘅值），唔好用舊嘅 3／6、6／9，否則「用量好滿」都不會觸發你以為嘅滿塊告警。
- **運維**：非緊急歸集／跨鏈再平衡避開 blob 高峰；已鎖價出金按 quote 收、超成本由 P&L 計，但廣播層設單筆 gas／L1 fee 上限以免一筆燒穿。成本帳用 receipt，唔用寫死 0.15。
- **回歸**：模擬 blob fee 5× 時報價上升或熔斷；L1 execution fee 不變時告警仍然要響；BPO 調參後唔使改代碼常數（讀配置／鏈上參數）。
