# 鏈上索引：自建掃鏈 vs The Graph（CEX 視角）

## 人話

CEX 要入帳，唔能靠「用戶話我打咗」——要自己睇鏈。兩條常見路：

1. **自建 indexer**：接 RPC／WS，跟 block、解 `Transfer`／合約事件，寫入自己 DB，再對帳入帳。
2. **The Graph 類托管索引**：用 subgraph（GraphQL）訂閱已索引好嘅事件／實體，少寫掃鏈管道，但信任同 SLA 喺第三方。

類比：你熟悉嘅「Kafka consumer 自己消費 vs 用現成數據倉庫／API」。CEX 充值路徑多數 **自建**（可控、可對帳、可 rollback reorg）；公開 DApp 多用 Graph 快速出查詢層。

## 面試短答

- 核心問題：鏈係 append-only log，業務要 **按地址／按合約／按事件** 查 → 需要物化索引
- 自建：`eth_getLogs`／subscribe + 游標（block 水位）+ 解碼 ABI + 冪等寫入；要處理 reorg（確認數／回滾水位）
- The Graph：寫 subgraph manifest + mapping，託管／去中心化 indexer 提供 GraphQL；適合前端／分析，唔適合把「最終入帳真相」完全外包
- CEX 實務：充提對帳、熱錢包監控、風控通常自建；Graph 可用於內部儀表／非關鍵查詢，但入帳閘要自己信自己的掃鏈結果
- 取捨：自建＝運維＋RPC 成本＋正確性責任；Graph＝快上線＋查詢方便，但延遲／分叉／供應商風險要評估

## 常見追問＋答案

Q：點解 CEX 充值很少「只信 The Graph」？  
A：入帳係資金最終性。Graph 有索引延遲、可能分叉未對齊你司確認策略、供應商／去中心化 indexer 可用性問題。合規同對帳要能重放自己的 block 水位同原始 logs。

Q：自建掃鏈最易踩咩坑？  
A：漏塊、`getLogs` 範圍過大被拒、只訂閱唔補歷史、reorg 未回滾、同一 tx 重放導致雙重入帳、多鏈／多合約配置錯（chainId×contract）。

Q：The Graph 同「自己跑 archive node + 自寫 ETL」點比？  
A：Graph 抽象咗索引同 GraphQL；自建 ETL 完全可控 schema／延遲／權限。高吞吐 CEX 常見：輕量 RPC + 自建流水線；研究型／多協議儀表先考慮 Graph。

Q：確認數同 indexer 水位點配合？  
A：indexer 可先寫「未最終」狀態；業務入帳只認 `blockNumber <= head - N` 或等 finality。reorg 時按共同祖先回滾 DB，再前進——同你做過嘅 RPC／finality 題同一套路。

## 小練習

題：你們自建 ERC-20 充值掃鏈，游標停在 block 1000。此時 chain head=1012，產品要 12 確認。用戶在 block 1001 打咗一筆 Transfer 到充值地址。系統應否立刻入帳？若之後 1001–1010 被 reorg 掉，應點設計？

參考答案：不應立刻最終入帳。可先記錄 `seen@1001`，等到 `head >= 1001+12`（或你們嘅 finality 規則）先標 `credited`。若 reorg 令 1001 消失：水位回滾到共同祖先，刪除／作廢未最終記錄，禁止只靠 txHash 永不回滾；已錯誤入帳要走凍結提現＋對帳補償（同 bridge／reorg 題一致）。
