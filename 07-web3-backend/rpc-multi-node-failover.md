# 多 RPC 節點 Failover 與一致性

## 人話（從你熟的 CEX）

CEX 撮合／DB 都有主從、讀寫分離同 failover。鏈上一樣：你唔會只靠一個 RPC 節點（自建 + Infura／Alchemy／QuickNode 等多家）。問題係：**唔同節點睇到嘅鏈頭唔同步**——有啲落後幾塊，有啲喺短暫分叉另一邊，`pending` 狀態（mempool）更加每個節點唔同。

類比：讀 MySQL 從庫有複製延遲；你唔會用從庫去決定「餘額夠唔夠扣」。鏈上 RPC 就係一堆「延遲唔一、偶爾分叉」嘅從庫。

## 面試短答

- **健康檢查**：每個節點睇 `eth_blockNumber` 同集群最高值嘅落後（lag）、錯誤率、延遲；lag 超門檻（例如 > 3 塊）踢出讀池。
- **按用途分級**：
  - 充值入帳：用 `finalized`／確認數水位，以 block hash 對照，最好 2 個獨立供應商一致先 credit（quorum）。
  - 出金 nonce／pending：**sticky** 去同一個（自建）節點，避免 A 節點有 pending、B 節點冇而誤判可以開新 nonce。
  - 廣播：可以同時送多個節點（同一 signed tx 同 hash，重複送無害）。
- **單調性守衛**：游標唔可以因為切去落後節點而「倒退」；讀到比已處理高度低嘅結果要忽略或等待。
- **Reorg vs 節點不一致**：同一高度兩個節點 hash 唔同，先等確認／對第三方，唔好即刻 rewind 帳本。

## 常見追問（連答案）

**Q：點解唔可以 round-robin 所有請求？**  
A：連續請求落去唔同節點會睇到「時間倒流」：先喺 A 讀到 block 100 有你嘅充值，下一次喺 B（落後）查 receipt 返 null，系統以為 tx 消失／被 reorg。對狀態有依賴嘅流程（掃塊游標、nonce、確認追蹤）要 sticky 或帶 `blockHash` 查詢，唔好純 round-robin。

**Q：供應商返錯數據（壞節點）點防？**  
A：關鍵寫帳路徑做 **交叉驗證**：入帳前用第二家節點以同一 `blockHash` 讀 receipt／log；大額出金前餘額同 nonce 雙源比對。長期記錄每家供應商嘅不一致次數，作為踢出依據。

**Q：主節點掛咗，切換時最易出咩事故？**  
A：(1) nonce：新節點 mempool 冇你之前嘅 pending → `pending nonce` 偏低 → 重用 nonce 或誤以為要重發；正確做法係以本地 nonce 帳 + 已廣播 signed tx 重新廣播去新節點。(2) 掃塊游標：新節點落後 → 等佢追上，唔好倒退游標。(3) WebSocket 訂閱斷線 → 用 `eth_getLogs` 按游標補漏。

## 小練習

**題：** 你有自建節點 N1、供應商 P1、P2。某時刻 N1 頭=1,000,050，P1=1,000,049，P2=1,000,041。充值服務游標已處理到 1,000,045；出金服務要為熱錢包取 nonce。N1 忽然斷線，系統應點分配流量？

**參考答案：**  
P2 落後 9 塊，超 lag 門檻 → 踢出讀池（唔好用佢推進或回退游標）。充值掃塊切去 P1，由 1,000,046 繼續；入帳仍等確認／finalized 水位，並可用 P2 追上後做第二源核對 hash。出金：唔好盲信 P1 嘅 `pending nonce`（佢 mempool 可能冇 N1 收過嘅 pending tx）——以本地 nonce 帳為準，把已簽未確認嘅 tx 重新廣播到 P1（同 hash 無害），確認 P1 `pending nonce` 追上之後先分配新 nonce。N1 恢復後先追平高度再放返入池。
