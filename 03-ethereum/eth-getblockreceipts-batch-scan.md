# eth_getBlockReceipts：一次攞晒成塊嘅 receipt（批量掃塊）

## 人話（從你熟的 CEX）

掃塊入帳有兩種讀法：

- `eth_getLogs`：叫節點「幫我篩」——只返指定合約／topic 嘅 log。好似 SQL 加 `WHERE`，返得少、快。
- `eth_getBlockReceipts(block)`：叫節點「成塊所有交易嘅結果（receipt）一次過俾我」。好似 `SELECT *` 成個 partition，自己喺服務端篩。

以前要逐筆 `eth_getTransactionReceipt`（一塊 200 筆就 200 次 RPC），而家一次就攞齊。

## 面試短答

- **係乜**：輸入 block number／tag／**block hash**，返該塊每筆 tx 嘅 receipt（`status`、`from`、`to`、`logs`、`effectiveGasPrice`、`contractAddress`…）。主流客戶端（Geth、Erigon、Nethermind、Besu）同大供應商都支援，但要逐條鏈／逐家確認。
- **幾時用 getBlockReceipts**：
  1. 要知**每筆 tx 成功定失敗**（例如核對自己一批出金；失敗 tx 冇 log，`getLogs` 永遠睇唔到佢）。
  2. 監控幣種／合約好多，`getLogs` 地址過濾變得好長，不如一塊全攞自己 match。
  3. 想**以 block hash 原子咁處理一塊**：同一個 hash 嘅 receipts 一致，方便做 reorg 對照。
- **幾時用 getLogs**：只關心少數合約嘅 `Transfer`、想慳頻寬／計費。主網一塊 receipts 可以好大，全掃成本高好多。
- **盲點**：receipt **冇 `value`**。Native ETH 頂層轉帳要睇 `eth_getBlockByNumber(n, true)` 入面嘅 tx；合約內部轉 ETH（internal transfer）要 trace。

## 常見追問（連答案）

**Q：點解要用 block hash 而唔係 block number 去攞？**  
A：用 number 嘅話，兩次請求之間可能發生 reorg，或者 failover 去另一個節點睇到另一條分支，你會「半塊舊、半塊新」。先 `getBlockByNumber` 攞 hash，再用 **同一個 hash** 攞 receipts，並記低 hash 入游標；之後確認時對 hash，唔同就 rewind。

**Q：一塊 200 筆交易，getBlockReceipts 同逐筆 getTransactionReceipt 有咩分別？**  
A：結果內容一樣，但 1 次 vs 200 次往返：延遲、限流（rate limit）、部分失敗處理都簡單好多。代價係單次 response 大，要設好 timeout／body size，並準備供應商唔支援時 fallback 返逐筆或 `getLogs`。

**Q：用 receipts 掃 ERC-20 充值，同 getLogs 結果應該一樣？**  
A：成功 tx 嘅 log 應該一樣（getLogs 本身就係由 receipts 篩出嚟）。差別係 receipts 仲會見到 `status=0` 嘅失敗 tx——有用嘅地方係：用戶話「我轉咗」但其實 revert，客服可以即刻答；但入帳只認 `status=1` 嘅 `Transfer` log，同埋要 check `log.address` 係白名單合約。

## 小練習

**題：** 你負責 USDT／USDC／另外 30 隻 ERC-20 嘅充值，同時要確認熱錢包每批 50 筆出金有冇成功。現時用 `eth_getLogs` 掃充值，出金逐筆 `getTransactionReceipt` 輪詢，RPC 量好大兼成日撞 rate limit。你會點改？

**參考答案：**  
改為「每塊一次」：先 `eth_getBlockByNumber(n, false)` 攞 hash，再 `eth_getBlockReceipts(hash)`。服務端一次過做兩件事：(1) 篩 `status=1` 且 `log.address` 喺 32 隻白名單合約、topic0 係 `Transfer`、`to` 係充值地址嘅 log → 入「待確認」；(2) 用 tx hash 對出金表，`status=1` 標成功、`status=0` 標失敗走失敗流程（唔好盲目重發）。游標存 `(number, hash)`，確認數／finalized 到先入帳，hash 對唔上就 rewind。Native ETH 充值另外睇 block 內 tx 嘅 `value`（內部轉帳靠 trace）。保留 fallback：供應商唔支援或 response 太大時，退返 `getLogs` + 逐筆 receipt。
