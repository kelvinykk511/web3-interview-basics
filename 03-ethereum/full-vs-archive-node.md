# Full vs Archive 節點（歷史狀態／debug／深掃）

## 人話（從你熟的 CEX）

CEX 掃鏈同排查，唔止「而家嘅餘額」——要查「三個月前嗰個 block 嘅 `eth_call`」「舊 tx 嘅 `debug_traceTransaction`」「一年前到而家嘅 `eth_getLogs`」。

類比：全量 DB 主庫（可寫近期）vs 帶完整歷史快照／時間旅行查詢嘅倉庫。**Full node** 跟到最新頭、保留足夠近的狀態去驗證新塊；**Archive node** 額外保留**每一個歷史 block 高度**嘅完整 world state，先可以對任意舊高度做 state 讀同重放。

面試常問：點解公共免費 RPC 一查舊 block 就失敗？多數因為你打去咗 non-archive。

## 面試短答

- **Full**：同步 + 驗證鏈；可服務近期 RPC；**唔保證**任意舊高度嘅 `eth_call`／`eth_getBalance(addr, blockTag)`／`debug_trace*`／storage 讀。
- **Archive**：在 full 之上保留歷史 state（磁碟大幾個數量級）；支援 `blockNumber` 指定舊高度嘅 state 類方法同多數 historical trace。
- **Logs 唔等於 archive**：`eth_getLogs` 主要靠 receipt／log 索引；有啲 full 實作仍可掃較長 log 範圍，但「舊高度 state」同「深歷史 trace」仍要 archive（或第三方 archive API）。
- **CEX 實務**：線上充提熱路徑用低延遲 full／專用讀寫池；對帳、事故屍檢、合規回溯走 **archive 池**（自建或付費 Alchemy／QuickNode archive）。`debug_*` 更要隔離，唔同用戶流量搶 CPU。
- **pruned／semi-archive**：客戶端可配置「保留最近 N 日 state」——面試講清「我哋 SLA 要幾耐歷史」，再選節點規格，唔好只背名詞。

## 常見追問（連答案）

**Q：`eth_getBlockByNumber`／receipt 係咪一定要 archive？**  
A：唔一定。區塊頭、交易、receipt 多數 full 都有（鏈數據本身）。缺嘅係**歷史執行後 state**（舊 `balanceOf`、舊 storage、對舊 tx 重放 trace）。分清「鏈數據」同「state」。

**Q：點解 `eth_call` 唔帶 blockTag 好似得，一加舊高度就 `missing trie node`？**  
A：默認 `latest` 用當前 state；舊高度要該高度嘅 trie。節點 prune 咗就冇。CEX 預檢出金用 `latest`（或 `pending`）即可；審計／糾紛「當時合約係咪 paused」要用 archive + 明確 blockTag。

**Q：L2 同 ETH 主網一樣分 full／archive 嗎？**  
A：概念類似，但實作同供應商產品名唔同；有啲 L2 歷史短、有啲要專用 archive。多鏈矩陣要標：每條鏈「熱路徑 RPC」vs「歷史／trace RPC」，同昨天講嘅 debug 可用性一齊維護。

**Q：自建 archive 好重，有冇折衷？**  
A：常見：(1) 熱數據自建 full + 冷查詢買 archive API；(2) 只對白名單合約／地址做自建索引物化，少打歷史 state；(3) 事故時臨時開 archive 實例。入帳真相仍應靠自己 indexer 水位，唔好每次入帳都打 archive `eth_call`。

## 小練習

**題：** 用戶投訴「兩個月前充值多入咗」，客服要你證明「當時 `balanceOf(充值地址)` 同 Transfer log 一致」。你只有 full node 同現有 indexer DB，會點做？邊步一定要 archive？

**參考答案：**  
1. 先信自建 indexer：用 `txHash+logIndex` 撈當時寫入嘅 log 解碼、確認數、入帳金額——呢啲係你們對帳真相。  
2. 用 full／任何有 receipt 嘅節點重拉該 tx receipt，核對 log 內容（唔一定要 archive）。  
3. 若要「該 block 高度執行 `balanceOf`」或重放內部 call 證明 fee-on-transfer 實收——**要 archive**（或當時 indexer 已存嘅 `balanceOf` 快照）。  
4. 回覆客服：以 indexer 冪等記錄 + receipt log 為準；缺 archive 就唔好臨時發明一個「現在 balance 倒推兩個月」嘅數。長期：對高風險幣種喺入帳時持久化「掃到時 balance 增量」。
