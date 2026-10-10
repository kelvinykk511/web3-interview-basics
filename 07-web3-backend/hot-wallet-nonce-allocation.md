# 熱錢包 nonce 分配（出金併發）

## 人話
EOA 每筆交易要用連續 nonce（0,1,2…）。好似銀行櫃位叫號：中間跳咗一個號（nonce 5 未上鏈），6、7、8 全部卡住。CEX 出金併發高，多個 worker 同時發，最易撞號或留洞。

## 面試短答
- 每個熱錢包地址一個「nonce 分配器」：DB 行鎖 / Redis 原子 INCR / 單線程 actor，**唔好每次問節點 getTransactionCount("pending")**（多實例會撞）。
- 分配 nonce 同簽名、持久化（nonce, rawTx, txHash）喺同一事務落庫，**先落庫再廣播**，重啟可重播。
- 監控「洞」：已分配但未上鏈最小 nonce 卡住 → 用同 nonce 提 gas 替換或發 0 值自轉填洞。
- 吞吐唔夠 → 多個熱錢包地址分片（按 withdrawalId hash），而唔係一個地址硬頂。

## 常見追問（連答案）
1. **廣播失敗點算？** 唔好回收 nonce 畀第二單（可能其實已入 mempool）。用同一 rawTx 重播；確定被拒（如 nonce too low = 已有交易用咗）再對帳。
2. **nonce too low 代表咩？** 該 nonce 已經被某筆交易上鏈——查返係咪自己嗰筆（txHash 一致）定被替換咗，按實際上鏈交易更新出金狀態。
3. **點解唔每次用 pending nonce？** pending 視圖各節點唔一致、mempool 會丟交易，多 worker 同時讀會拎到同一個號。
4. **一單出金取消？** nonce 已分配就唔可以「刪」，要用同 nonce 發一筆 0 值自轉（更高 gas）佔位，否則後面全卡。

## 小練習
題：nonce 10 出金因 gas 太低卡咗 2 小時，11–15 都排隊。你點處理？
答：用 nonce 10 發替換交易（同內容、maxFeePerGas/priorityFee 各至少 +10%），記錄新 txHash 同舊 hash 關聯；兩個 hash 都監聽，邊個上鏈就以邊個為準；10 一上鏈，11–15 自然順序打包。事後檢討 gas 估算策略同卡單告警閾值。
