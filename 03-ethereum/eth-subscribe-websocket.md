# eth_subscribe（WebSocket）— CEX 充值實時監聽

## 人話（從你熟的 CEX）

CEX 充值掃描器要「塊一出就知有冇入金」。HTTP 輪詢 `eth_getBlockByNumber`／`eth_getLogs` 穩但有延遲；WebSocket 嘅 `eth_subscribe` 可以訂閱 **newHeads**（新塊頭）同 **logs**（符合 filter 嘅事件），接近推送。

人話：訂閱似 Kafka consumer 收推送，但鏈上節點 **會丟、會斷、會分叉**——所以生產唔可以只靠訂閱；一定要「訂閱催促 + 按塊補掃」雙保險，再用 `chainId + txHash + logIndex` 冪等入帳。

## 面試短答

- `eth_subscribe` 走 **WebSocket**（唔係普通 HTTP JSON-RPC 單次 request）。常見 subscription：
  - `newHeads`：每個新區塊 header（催你去掃／對水位）
  - `logs`：帶 `address`／`topics` filter，推符合條件嘅 log（例如某 ERC-20 嘅 `Transfer`）
  - （較少用）`newPendingTransactions`：pending hash，嘈、易丟，CEX 入帳一般唔靠佢
- 回傳係 **推送流**；斷線、節點 failover、proxy idle timeout 都會漏事件。
- CEX 標準架構：WS 訂閱做 **低延遲觸發**；`lastScannedBlock` 游標 + 定時／按塊 `eth_getLogs` 做 **至少一次補掃**；業務層冪等。
- 訂閱 `logs` 時 filter 要夠窄（熱門 token 合約 + `Transfer` topic0 + 可選 `to` 地址集合），否則流量爆炸、後端塞爆。

## 常見追問（連答案）

**Q：點解唔可以「只開 logs 訂閱、唔補掃」？**  
A：WS 唔保證 exactly-once：斷線期間漏推、節點 reorg 視角切換、訂閱 ID 失效、負載均衡換連線。漏咗就永遠唔入帳。補掃用區塊區間 `fromBlock–toBlock` 重放，先保證 eventually consistent。

**Q：newHeads 同 logs 訂閱點配合？**  
A：常見：`newHeads` 推進「鏈頭水位」→ 對每個新高度（或批量）用 `eth_getLogs` 掃；或同時訂 `logs` 做快路徑，仍用 newHeads／定時對齊游標。入帳確認數仍跟 header 高度同 reorg 水位，唔單信一條 push。

**Q：topics 點寫 Transfer？**  
A：`Transfer(address,address,uint256)` 嘅 topic0 = `keccak256` 簽名。indexed 參數：`from`=`topics[1]`、`to`=`topics[2]`（ERC-20）；金額喺 data。CEX 充值常 filter：`address=token`、`topics=[Transfer, null, userDepositAddress]`（視 RPC 語法用 `null` 做 wildcard）。注意有啲非標準 token 事件唔跟呢個布局。

**Q：reorg 時訂閱會點？**  
A：節點可能推「同一高度另一個 hash」嘅 newHead，或 removed logs（視客户端／節點）。狀態機要把未達確認數嘅 pending 入帳標成可回滾；達標確認／`safe`／`finalized` 先轉可用餘額。唔好喺「剛收到一條 log」即刻當不可逆入帳。

## 小練習

**題：** 你哋 ETH 充值服務用 `logs` 訂閱 USDT `Transfer` 到熱錢包地址。某日 WS 靜默斷線 3 分鐘無人發現，期間有 5 筆真實入金。系統應點設計先保證用戶最終上分、又唔會重複入帳？

**參考答案：**  
1. **健康檢查**：訂閱心跳／newHeads 超時無訊息 → 告警 + 自動重連 + 標記 `catchup_required`。  
2. **補掃**：用持久化 `lastScannedBlock`（或 lastSafeBlock）從斷線前水位掃到當前 head−N；對每條 log 用 `chainId+txHash+logIndex`（或等效）做唯一鍵 upsert。  
3. **確認數**：補掃到嘅 tx 一樣走 `seen → confirming → credited`，reorg 可回滾未達標。  
4. **唔依賴**「重連後訂閱會重放歷史」——多數唔會；歷史只靠 `eth_getLogs`／掃塊。  
5. 可選：多 RPC 源交叉，避免單一 provider 靜默丟推送。
