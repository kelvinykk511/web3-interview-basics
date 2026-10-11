# 跨鏈橋資產充值

**人話**：Arbitrum 上嘅 USDC 同以太坊 USDC 唔一定係同一樣嘢——有原生（Circle 發）同橋接版（USDC.e）。

**面試短答**：資產用 (chainId, contract) 做 key，原生同橋接分開上架/分開餘額或明確映射；只認白名單合約；L2 充值按該鏈最終性（L2 soft vs L1 batch 確認）；橋本身風險（被黑、脫錨）要可一鍵暫停。

**常見追問**
- 用戶由 L1 經官方橋存去自己交易所 L2 地址？→ 到賬係 L2 上 mint/轉賬事件，照常掃 L2 Transfer；L1 側唔入帳。
- Optimistic rollup 提現 7 日？→ 只影響 L2→L1 官方橋；交易所 L2 充值通常按 L2 確認 + 等 L1 batch 提交（safe）。
- 用戶將 USDC.e 打去只支持原生 USDC 嘅地址？→ 唔自動入帳，入人工找回流程（同錯幣/錯鏈處理）。
- 橋被黑點算？→ 監控橋合約/新聞，暫停該橋接資產充提，避免收入無擔保代幣。

**小練習**：點樣避免「同名 USDC 假幣」入帳？
**參考答案**：唔睇 symbol/name，只認 chainId+合約地址白名單；Transfer 事件嘅 emitter 地址必須等於白名單合約。
