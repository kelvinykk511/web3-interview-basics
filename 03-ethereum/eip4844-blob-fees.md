# EIP-4844 Blob Fee — 點解 L2 忽然平咗、CEX 要分開兩本帳

## 人話

以前 L2 把大量交易數據當 **calldata** 寫上 L1，跟普通 execution gas 搶同一條費市場，貴。  
EIP-4844（proto-danksharding）引入 **blob**：一種「掛喺區塊旁邊」嘅大數據位，有**獨立嘅 blob gas／blob base fee** 市場，專門畀 rollup 交數據。

對 CEX 工程師：你提幣／充值喺 Arbitrum、Optimism、Base 等 L2 時，用戶感覺「Gas 平咗」，背後往往係 **L2 執行費 + 攤分後嘅 L1 數據可用性（blob）成本**。你嘅多鏈 fee oracle **唔可以再假設只有一條 EIP-1559 gas**；L2 要分開估「L2 execution」同「數據／blob 傳導成本」（有時反映喺 L2 嘅 `l1Fee`／`blob` 相關欄位）。類比：CEX 撮合本身一條路，清算／結算另一條路——兩邊擁塞唔一定同步。

## 面試短答

4844 讓區塊可攜帶 blob，blob 有獨立 base fee，目標係令 rollup 數據刊登遠平過 calldata。  
Blob 數據**唔會**永久留喺執行層狀態（約數週後可被淘汰），夠 DA／驗證用但唔係永遠可從 L1 合約直接讀。  
CEX 多鏈成本模型：L1 出金繼續看 1559 gas；L2 出金要讀該鏈嘅 gas price oracle **加上** L1 data／blob 成分（各鏈字段名唔同），再折算成提幣手續費；擠兑時 blob fee 同 L2 gas 可能分開飆升。

## 常見追問（連答案）

**Q: Blob 同 calldata 最大分別？**  
A: Calldata 進執行層、合約可讀、較貴；blob 主要畀共識／DA 用、執行層合約**一般讀唔到 blob 原文**，但便宜得多，適合 rollup batch。面試加一句：blob 有獨立 fee market（`blobGas`／blob base fee）。

**Q: CEX 後端要直接發 blob tx 嗎？**  
A: 多數 CEX 出金只係喺 L2 發普通轉帳／合約呼叫，**唔使自己組 blob**；blob 係 sequencer／rollup 批次上 L1 時用。你要懂嘅是**成本來源同監控**：L2 提幣費突然升，可能係 L2 congestion，亦可能係 L1 blob fee 飆高傳導落嚟。

**Q: Blob fee 點樣「自動」調？同 1559 似唔似？**  
A: 似 1559 有 target：blob gas 用得超過目標 → blob base fee 升；低過目標 → 降。CEX 監控要睇 `excessBlobGas`／blob base fee，唔好只盯 L1 `baseFeePerGas`——兩條曲線可以完全唔同步。面試加一句：目標約每塊若干 blob，唔係無限塞。

**Q: 同「最終性／提款延遲」有無關係？**  
A: 正交。4844 主要改**數據刊登成本**；Optimistic／ZK 嘅挑戰窗、證明時間仍然決定 hard finality。CEX 確認數策略唔好因為「blob 平咗」就自動減確認——安全假設冇變。

## 小練習

**題：** 運維告警：過去 1 小時 Optimism 出金平均成本升 3 倍，但 L1 ETH transfer 估價幾乎冇變；同時 L2 `baseFee` 只微升。你點排查？會唔會即刻全面加 tip？

**參考答案：**  
先拆三個數：（1）該 L2 execution baseFee／tip，（2）該鏈 receipt／oracle 披露嘅 L1 data／blob 成分，（3）L1 `blobBaseFee`／`excessBlobGas`。若 (1) 微升但 (2)(3) 飆高，就係 **DA／blob 市場**傳導，唔係「tip 唔夠」。全面加 tip 解決唔到數據費；應調提幣手續費／暫停非緊急大額出金佇列節奏，並分開報警「執行擁塞」vs「blob 擁塞」。產品話術：L1 轉帳平 ≠ L2 出金一定平。
