# Event Log 索引陷阱（CEX 充值 Scanner）

> 銜接：[ERC-20 Transfer 事件](../04-smart-contracts/erc20-transfer-events.md)、[getLogs 游標](../03-ethereum/eth-getlogs-range-cursor.md)、[getBlockReceipts](../03-ethereum/eth-getblockreceipts-batch-scan.md)、[今日 reorg／確認數](./deposit-reorg-confirmation-depth.md)。今日專攻 **解碼同索引常見踩坑**：topics vs data、logIndex、漏 log、假事件、reorg removed。

## 人話（CEX 類比）

當你做支付回調驗簽：要驗 **欄位係咪對、係咪重複、係唔係預期商戶**。鏈上 log 一樣——睇到一條 `Transfer` 唔等於可以入帳。你要識：

1. **點過濾**（topics）vs **點讀金額**（data）  
2. **同一 tx 多條 log** 點分（logIndex）  
3. **漏掃／重掃／reorg** 點唔雙花、唔漏單  
4. **假合約／錯 chain／只有內部轉帳冇 event** 點擋

Indexer bug = 用戶投訴「有 txHash 但未到帳」或更糟「到咗兩次」。

## 面試短答

- **Log 結構**：`address`（emit 嘅合約）、`topics[0]` = event signature hash、indexed 參數進 `topics[1..]`、非 indexed 進 `data`。
- **Transfer**：`topics[0]=keccak256("Transfer(address,address,uint256)")`，`from`／`to` 多為 indexed（topics），`value` 喺 `data`（uint256 big-endian）。
- **過濾**：`eth_getLogs` 用 `address` + `topics`（例如 topic2 = 充值地址 left-pad 32 bytes）。**唔好**只拉全部再喺應用層海量過濾（慢、易超限）。
- **冪等**：`chainId + txHash + logIndex`（加 token 合約更穩）；同一 tx 可多筆 Transfer。
- **漏 log 來源**：WS 斷線、步長跳窗、只訂 tip 唔補歷史、節點 prune／限流回唔完整、錯 topic／錯合約。
- **假陽性**：同名 symbol 假幣、`Transfer` 簽名一樣但係垃圾合約、address poisoning（大氣額 + 塵額假地址）。
- **冇 event 嘅移動**：某啲內部 accounting、rebase、`destroyBlackFunds`、錯用 `eth_call` 當入帳——要 balance diff／對帳補位，唔係淨靠 log。
- **reorg**：subscription 可能推 `removed: true`；getLogs 重掃同一範圍要靠冪等 + 狀態回滾（見 reorg 筆記）。

## 常見追問（連答案）

**Q1：topics 同 data 有咩分別？解錯會點？**  
A：indexed 參數入 topics（最多再加 3 個 indexed；動態類型 indexed 只存 hash）。`value` 若被誤當 topic 去 filter 會永遠 match 唔到；若把 `to` 當 data 解碼，偏移一錯就入錯地址／金額。面試可講：先用 ABI 解碼，唔好手寫切 byte 除非你極熟 padding（address 喺 topic 係 32-byte left-padded）。

**Q2：點解冪等鍵一定要有 logIndex？**  
A：一筆 swap／聚合 tx 可以 emit 十幾條 Transfer。只用 `txHash` 會：要嘛只入第一筆漏其餘，要嘛重放時唔知更新邊條。CEX 充值通常仲要 match `to == 用戶充值地址` 同白名單 token。

**Q3：getLogs 回傳空，但 explorer 睇到 Transfer——常見原因？**  
A：（1）`fromBlock/toBlock` 窗口冇蓋住；（2）`address` 填錯（用咗用戶地址而唔係 token 合約）；（3）topic 編碼錯（地址冇 pad 到 32 bytes、大小寫無關但 hex 長度錯）；（4）節點唔係 archive／狀態唔齊；（5）你查緊錯 chainId 嘅 RPC；（6）reorg 後嗰條已經唔喺 canonical chain。排查：用同一 RPC 打 `eth_getTransactionReceipt` 睇 logs 原始內容，再對照 filter。

**Q4：只 subscribe logs，唔跑 getLogs 補掃，可以嗎？**  
A：生產唔建議。WS 會斷、進程會重啟、消息會丟。正確係 subscribe 催促近端 + **游標 getLogs／getBlockReceipts 做可靠底盤**；兩者寫入同一冪等表。

**Q5：`status=1` 但用戶話冇到帳？**  
A：可能：轉嘅係假 token；`to` 唔係該用戶充值地址（poisoning／轉錯）；fee-on-transfer 實際到帳少過 amount（要用 balance 前後差）；充值地址係合約但你只掃 EOA 列表；確認數未夠所以 UI 仲 confirming；或者你掃嘅係 Transfer，實際係原生 ETH（要分原生 vs ERC-20 兩條管線）。

**Q6：indexed 咗 string／dynamic 類型點解 match 唔到？**  
A：dynamic indexed 喺 topic 存嘅係 **hash**，唔係原文。唔可以靠 topic 還原原文，亦唔可以用明文當 topic filter。Transfer 嘅 address／uint 唔受呢個問題影響；自己設計 event 時要小心。

## 小練習（題目後即附參考答案）

**題：** 充值 scanner 配置：只 listen 白名單 USDT 合約嘅 `Transfer`，`topics` filter `to=用戶地址`。上線後三個 ticket：(A) 用戶貼嚟 explorer 連結，tx 成功且有 Transfer 入佢地址，系統無單；(B) 同一 txHash 系統入咗兩次帳；(C) 一筆已顯示「確認中」嘅單突然消失。分別最可能咩原因？點修？

**參考答案：**  
(A) 常見：RPC／filter 用錯 chain；topic 地址 padding 錯；游標跳窗漏塊；只 subscribe 斷線未補掃；或者實際係 fee-on-transfer／代理合約多一跳，`to` 先入中間合約再轉入——filter 太死會漏。修：receipt 對照 raw logs、補 getLogs 游標、必要時跟 balance diff、多跳要產品定義清。  
(B) 冪等鍵唔完整（只用 txHash、或重放時冇 unique 約束）；或者 reorg 重掃當新單。修：DB unique `(chainId, txHash, logIndex)`，upsert 語義。  
(C) reorg／`removed`：pending 被回滾。修：UI 區分 confirming vs credited；回滾只動未入可用；游標後退重掃（見 reorg 筆記）。
