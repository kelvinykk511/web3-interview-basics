# EIP-4844 Blob Fee — 點解 L2 忽然平咗、CEX 要分開兩本帳

## 人話

以前 L2 把大量交易數據當 **calldata** 寫上 L1，跟普通 execution gas 搶同一條費市場，貴。  
EIP-4844（proto-danksharding）引入 **blob**：一種「掛喺區塊旁邊」嘅大數據位，有**獨立嘅 blob gas／blob base fee** 市場，專門畀 rollup 交數據。

對 CEX 工程師：你提幣／充值喺 Arbitrum、Optimism、Base 等 L2 時，用戶感覺「Gas 平咗」，背後往往係 **L2 執行費 + 攤分後嘅 L1 數據可用性（blob）成本**。你嘅多鏈 fee oracle **唔可以再假設只有一條 EIP-1559 gas**；L2 要分開估「L2 execution」同「數據／blob 傳導成本」（有時反映喺 L2 嘅 `l1Fee`／`blob` 相關欄位）。

## 面試短答

4844 讓區塊可攜帶 blob，blob 有獨立 base fee，目標係令 rollup 數據刊登遠平過 calldata。  
Blob 數據**唔會**永久留喺執行層狀態（約數週後可被淘汰），夠 DA／驗證用但唔係永遠可從 L1 合約直接讀。  
CEX 多鏈成本模型：L1 出金繼續看 1559 gas；L2 出金要讀該鏈嘅 gas price oracle **加上** L1 data／blob 成分（各鏈字段名唔同），再折算成提幣手續費；擠兑時 blob fee 同 L2 gas 可能分開飆升。

## 常見追問（連答案）

**Q: Blob 同 calldata 最大分別？**  
A: Calldata 進執行層、合約可讀、較貴；blob 主要畀共識／DA 用、執行層合約**一般讀唔到 blob 原文**，但便宜得多，適合 rollup batch。面試加一句：blob 有獨立 fee market（`blobGas`／blob base fee）。

**Q: CEX 後端要直接發 blob tx 嗎？**  
A: 多數 CEX 出金只係喺 L2 發普通轉帳／合約呼叫，**唔使自己組 blob**；blob 係 sequencer／rollup 批次上 L1 時用。你要懂嘅是**成本來源同監控**：L2 提幣費突然升，可能係 L2 congestion，亦可能係 L1 blob fee 飆高傳導落嚟。

**Q: 點監控先算專業？**  
A: 分指標：`L2 baseFee/tip`、該鏈披露嘅 `l1SecurityFee`／`eth_getBlockByNumber` 上 blob gas used／excess blob gas、再加自家 pending 出金隊列時延。報警分開「執行擁塞」vs「DA／blob 擁塞」，避免只加 tip 卻解決唔到數據費問題。

**Q: 同「最終性／提款延遲」有無關係？**  
A: 正交。4844 主要改**數據刊登成本**；Optimistic／ZK 嘅挑戰窗、證明時間仍然決定 hard finality。CEX 確認數策略唔好因為「blob 平咗」就自動減確認——安全假設冇變。

## 小練習

**題：** 某日 Base 提幣費齊齊上漲，但 Ethereum L1 普通 gas 平靜。你會先查邊兩個數？點解釋畀產品聽？

**參考答案：**  
先查（1）該 L2 嘅 execution base fee／擁塞，（2）L1 **blob base fee／excess blob gas**（或該 L2 顯示嘅 L1 data fee 成分）。若 L1 gas 平但 blob fee 高，就解釋：rollup 今日主要係**數據可用性／blob 市場**貴，唔係 ETH 轉帳本身貴；提幣費上調係傳導成本，唔單係「加 tip 就可以」。
