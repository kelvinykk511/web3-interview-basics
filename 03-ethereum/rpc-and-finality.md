# RPC 與最終性（給交易所後端）

## 人話

**RPC 節點** = 你查鏈上狀態、廣播交易嘅網關（類似你調支付渠道 API）。  
**最終性** = 「呢筆仲會唔會被重組掉」。CEX 用確認數／最終性規則決定幾時畀用戶上分、幾時當提現成功。

## 面試短答

- 讀鏈：`eth_getBlockByNumber`、`eth_getLogs`、receipt；寫鏈：`eth_sendRawTransaction`
- 監聽：輪詢掃塊 vs websocket 訂閱 newHeads／logs；要處理斷線補掃
- 可信 RPC：第三方單點、限流、回傳滯後／分叉視角 → 多源核對或自建節點
- 重組：已「看到」嘅區塊可能作廢 → 待確認入帳要能回滾；達標確認數先入可用餘額

## 常見追問＋答案

Q：點解充幣要等 N 個確認？  
A：確認愈多，被更深重組踢走嘅機率愈低。唔同鏈最終性模型唔同（PoW 概率最終 vs 一啲 PoS 有明確 finality）。N 係產品／風控參數，唔係魔法常數。

Q：RPC 返回「成功」但之後 reorg？  
A：RPC 只係某一個節點視角嘅當前鏈頭。短確認時可能仲喺 transient fork。狀態機應：`seen → pending_confirmations → credited`，reorg 時把未達標嘅 pending 回滾。

Q：掃塊同訂閱邊個穩？  
A：訂閱快但會丟事件；生產常見係「訂閱催促 + 定時／按塊補掃」雙保險，並用游標（lastScannedBlock）保證至少一次處理 + 業務冪等。
