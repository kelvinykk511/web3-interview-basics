# Base Denim：200ms 出塊同 Flashblocks 退役（CEX 充值掃鏈）

> 銜接：L2 充值最終性見 [l2-deposit-finality.md](./l2-deposit-finality.md)，sequencer 風險見 [../03-ethereum/l2-sequencer-risk.md](../03-ethereum/l2-sequencer-risk.md)。今日講 **Base Denim**（Mainnet 已於 2026-11-05 18:00 UTC、即香港時間 2026-11-06 02:00 啟動）：由「每 2 秒一個 canonical block + Flashblocks 預確認」改成 **每 200ms 一個完整 canonical block**，Flashblocks 退役。出塊快咗 **唔等於** 最終性升級。

官方說明：[Denim overview](https://docs.base.org/upgrades/denim/overview)、[200ms Native Blocks](https://docs.base.org/upgrades/denim/200ms-blocks)、[Migrate From Flashblocks](https://docs.base.org/upgrades/denim/migrate-from-flashblocks)、[Transaction Finality](https://docs.base.org/specifications/transactions/transaction-finality)。

## 人話（從你熟的 CEX）

以前 Base 大約兩秒先出一個「完整區塊」，中間用 Flashblocks 俾你睇 pending 預覽（未封塊）。Denim 之後，鏈自己就係每秒五個完整區塊：每個都有自己嘅 block number、hash、state root、receipts，同原本一樣行 **unsafe → safe → finalized**。Flashblocks 嘅 WebSocket／訂閱（例如 `newFlashblocks`、`pendingLogs`，同 `wss://mainnet.flashblocks.base.org/ws`）已經冇數據，稍後會斷線。

類比撮合引擎：以前係「2 秒一次正式成交回報 + 中間推送草稿」；而家係「每 200ms 一次正式成交回報」。回報密咗，但大額入帳仍然要等結算完成（batch 上 L1、再等 L1 finalized），唔好因為「200ms 已經有 receipt」就當不可逆。

另外，EVM 嘅 `block.timestamp` **仍然係整秒**。同一秒可以有 5 個區塊共用同一個 `timestamp`，毫秒部分放喺系統交易同可選嘅 RPC 欄位 `timestampMs`／`blockTimestampMs`。如果你用「區塊時間」做唯一鍵，或者用「等 N 個區塊」當幾分鐘，而家會全部錯。

## 面試短答

1. **出塊快咗，最終性層級冇變。** 官方而家寫嘅大約時間：L2 入塊約 200ms；batch 上 L1 約 2 分鐘；L1 batch 最終確定約 20 分鐘（2 個 epoch／64 個 L1 blocks）。大額充值仍然用 `safe`／`finalized`（或你哋既定金額分級），唔好用「已經有 receipt／等滿 N 個 L2 block」代替。由 L2 提返 L1 嘅挑戰窗同 Denim 無關（Beryl 之後單證明約 5 日，TEE+ZK 雙證明快路約 1 日）。
2. **`block.number` 嘅牆鐘速度加快 10 倍。** 以前 2 秒一格，而家 200ms 一格。用「確認數」當時間會嚴重低估：例如以前大約一分鐘嘅塊數，而家大約 6 秒就走完。`BLOCKHASH` 仍然只暴露最近 256 個塊，牆鐘窗口由大約 8.5 分鐘縮到大約 51 秒；EIP-2935 歷史合約仍然係 8,191 項，覆蓋時間同樣縮到約十分之一。EIP-4788 beacon roots 按 timestamp 存，窗口不變。掃鏈 `getLogs` 如果寫死「回看 3000 塊」，以前大約 100 分鐘，而家只有大約 10 分鐘，會漏單。
3. **唔好再用 Flashblocks／`pending` 當預確認。** `"pending"` 呢個 tag 仲接受，但官方寫明佢而家 resolve 做 **最新 canonical block**，不再暴露 Flashblocks 預確認狀態。Receipt 只會喺交易進入 200ms canonical block 之後先有，冇「預確認 receipt」。自家節點要 `base/base` v1.5.1 或更新（Mainnet），並清走舊 Flashblocks 旗標，以免未來版本啟動失敗。
4. **時間欄位要分清楚。** `timestamp` 維持 Unix 秒，同一秒可以對應五個 `block.number`。需要亞秒級排序就用官方派生嘅 `timestampMs`（毫秒 Unix；缺失就唔好自己用秒欄位估）。每個 Denim 區塊 `tx[0]` 仍係 L1 資訊 deposit，`tx[1]` 係 BaseTime 系統交易（毫秒部分只可以係 0／200／400／600／800），後面先至用戶交易。掃塊唔好把系統交易當成用戶入金。
5. **上架／合約風險：用 `block.number` 計時嘅合約會提早 10 倍到期。** Timelock、投票窗、每塊速率上限都會變。用 `block.timestamp` 嘅邏輯唔受影響。CEX 對 Base 上嘅金庫／鎖倉／治理代幣要評估呢項，唔好當純粹 UX 升級。

## 常見追問（連答案）

**Q1：200ms 就有 receipt，可唔可以當已確認入帳？**  
A：Receipt 只證明交易已被 sequencer 打進一個 canonical L2 block（大約即係官方講嘅「L2 inclusion」），仍然處於 unsafe 階段。官方強調呢層歷史上 reorg 極低，但政策上大額仍然應該等 `safe`（batch 已上 L1）或 `finalized`（L1 最終確定）。快係體驗，唔係把入帳標準放寬。

**Q2：掃鏈訂閱 `newHeads` 而家每秒五次，會唔會打爆消費者？**  
A：會，如果每個 head 都同步打 RPC、寫庫或做風控。要有背壓、合併處理、同用區塊 hash 做游標（唔好只靠 timestamp）。舊 Flashblocks WS 唔再有數據，若掃描器仲訂住舊 endpoint，表現會變成「好似冇充值」，要盡快切去 `eth_subscribe("newHeads")`／`logs`。

**Q3：`block.timestamp` 唔變，係咪時間相關邏輯都安全？**  
A：用秒級 timestamp 比較牆鐘時間嘅合約大致唔變；用塊數當時間、或者用 `(chainId, timestamp, txHash)` 當唯一鍵就會出事——同一秒有多個塊、甚至同一秒內多筆同樣 timestamp 嘅 log。冪等鍵仍然用 `(chainId, txHash, logIndex)`，必要時加上 block hash。橋／證明如果依賴短 `BLOCKHASH` 窗口，而家只有大約 51 秒，要重新評估。

## 小練習（題目 + 參考答案）

**題目：** Base 充值掃描器仍連住 `wss://mainnet.flashblocks.base.org/ws`，入帳規則係「見到 receipt 再等 30 個 L2 區塊就 credited」。另外用 `getLogs` 每次回看 1800 塊做漏單補掃（以前大約 1 小時），冪等鍵係 `(chainId, block.timestamp, txHash)`。Denim 之後會出現咩問題？點改？

**參考答案：**

- **Flashblocks WS 已冇數據**：補掃／即時掃描會靜默停擺。改用 canonical `newHeads`／`logs`，並確認節點已升級到官方要求版本。
- **「receipt + 30 塊」而家大約只係 6 秒**，仍然遠未到 L1 batch／finalized。應改回按 `safe`／`finalized`（或金額分級），唔好用 L2 確認數模擬時間。
- **回看 1800 塊而家只有大約 6 分鐘**，漏單窗口不夠。按牆鐘（或對應塊數 ×10）加長 range，並注意 `newHeads` 頻率同 `getLogs` 區塊範圍限制。
- **用 `block.timestamp` 做冪等會撞鍵**（同一秒五個塊）。改用 `(chainId, txHash, logIndex)`，reorg 用 block hash 回退。系統交易（`tx[0]`／`tx[1]`）唔入用戶充值。
