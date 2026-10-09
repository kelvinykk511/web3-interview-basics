# 充值入帳：Reorg、確認數與最終性（CEX Deposit Pipeline）

> 銜接：[rpc-and-finality](../03-ethereum/rpc-and-finality.md) 只講概念；[L2 finality](../03-ethereum/l2-finality-and-withdrawal.md) 講 L2 提款；[getLogs 游標](../03-ethereum/eth-getlogs-range-cursor.md) 講掃塊水位。今日講 **CEX 充值狀態機點樣擋 reorg、確認數點定、credited 之後仲可唔可以回滾**。

## 人話（CEX 類比）

你當支付渠道回調：銀行話「到咗」唔等於可以即刻放貨——有時會 chargeback。鏈上一樣：**RPC 見到塊 ≠ 永遠喺 canonical chain**。短時間可能分叉，舊塊被踢走（reorg），入面嗰筆 Transfer 當冇發生過。

所以充值唔好一步到位「見 log → 加可用餘額」。要用狀態機：

`seen（見到）→ confirming（等確認）→ credited（已入可用）`，必要時仲有 `held（風控暫扣）`。

確認數 N = 產品／風控參數：等 head 再前進 N 個塊，先當「夠穩」入帳。N 愈大愈安全、用戶等得愈耐——同支付「T+0 vs T+1」取捨一樣。

## 面試短答

- **Reorg**：本地／RPC 以為係主鏈嘅區塊，後來變成 uncle／被更深鏈取代；該塊內 tx／logs 失效。
- **確認數**：`confirmations = head - depositBlock + 1`（或 `head - depositBlock`，團隊要統一定義）；達標先由 confirming → credited。
- **最終性模型唔同**：PoW／一啲 L2 soft confirm 係**概率最終**（愈深愈穩）；Ethereum PoS 有 `justified`／`finalized` checkpoint（`eth_getBlockByNumber("finalized")`）。產品可以「確認數」或「等 finalized」或混合。
- **狀態機**：indexer 可超前寫 `seen`；**可用餘額**只喺達標後加。reorg 深度超過未入帳水位 → 回滾 pending；已 credited 要靠更深確認／finality 政策，盡量唔做「入咗又扣返」（用戶體驗同帳本災難）。
- **多 RPC**：唔同節點可能喺 tip 睇到唔同分叉 → 入帳用 quorum／sticky + 保守 head（見 [rpc-multi-node-failover](./rpc-multi-node-failover.md)）。
- **冪等鍵**：`chainId + txHash + logIndex`（原生幣可用 `txHash + ''` 或內部 index）；reorg 重放唔雙加。

## 常見追問（連答案）

**Q1：點解唔見到 Transfer 即刻入可用餘額？**  
A：tip 附近最易 reorg。即入可用 = 用戶可能已經交易／提現，之後 reorg 要倒扣——帳本、風控、客服全部爆。正確係先 `confirming`（UI 顯示「確認中 3/12」），達標先 `credited`。

**Q2：確認數點定？ETH 要 12、BTC 要 3 係咪標準答案？**  
A：唔係魔法常數。睇：（1）鏈嘅最終性模型同歷史 reorg 深度；（2）幣值同用戶層級（大額加確認）；（3）自家 RPC／自建節點滯後；（4）競品 SLA。ETH PoS 之後深 reorg 極罕，有團隊用較細 N 或跟 `finalized`；L2 soft confirm 可能 1 塊就顯示，但「可提現／可出金」另用更嚴規則。答面試要講「風險參數 + 鏈模型」，唔好背死數。

**Q3：已經 credited，之後發現 reorg 咗點算？**  
A：設計目標係 **credited 水位低過實際可能 reorg 深度**，所以生產上極少發生。真發生（極端、錯配 N、惡意深 reorg）：凍結相關帳戶、對帳差額入「異常損失／風控池」、人工調查；唔好靜靜倒扣已用咗嘅餘額。預防：入帳跟 `safe`／`finalized` tag、大額額外確認、多源 quorum。

**Q4：`latest` vs `safe` vs `finalized` 點用喺充值？**  
A：`latest` = tip，追新最快、reorg 風險最高，只適合寫 `seen`／催促 UI。`safe`（CL 定義下相對穩嘅頭）適合大多數自動入帳。`finalized` 最適合大額／可提現門檻。Scanner 可以掃到 `latest-ε`，但 **狀態推進到 credited** 用 safe／finalized／N-confirm 其中一套寫死喺配置。

**Q5：L1 同 L2、同係「12 確認」意思一樣嗎？**  
A：唔一樣。L2 塊係 sequencer 排嘅，soft confirm 快但要分清「sequencer 答應咗」vs「已經 batch 上 L1／過挑戰窗」。充值入帳可以用 L2 確認數；**允許提現到鏈外或大額出金**可能要等更強最終性（見 L2 finality 筆記）。面試講清楚產品動作綁邊層最終性。

**Q6：reorg 時游標同 DB 點回滾？**  
A：偵測：newHeads 父哈希對唔上、或 `eth_getBlockByHash` 變 null、或 subscription `removed: true`。做法：找到共同祖先，把 DB 裡 `blockNumber > ancestor` 且未達最終策略嘅存款標 `reorged`／刪 pending，游標退到 ancestor，再向前掃。已 credited 且策略認為不可逆嘅列，唔好自動刪——走異常流程。

## 小練習（題目後即附參考答案）

**題：** 你哋 ETH 充值配置係「12 確認後入可用」。某筆 USDT Transfer 喺 block 1005，當時 head=1010，狀態 `confirming`（確認數約 6，未達標）。之後發生 reorg：舊 1005 連同該 tx **唔再存在於 canonical chain**，新鏈從 1004 之後分叉。請寫：（1）偵測到之後狀態機點變；（2）用戶 UI／餘額點顯示；（3）點樣避免「未夠確認就入可用」。

**參考答案：**  
1. 偵測父鏈斷裂、`getBlockByHash(舊1005)` 失敗、或 receipt／log 消失（或 `removed: true`）→ 將該筆由 `confirming` 改 `reorged`（或刪 pending），游標退到共同祖先（≤1004），從新鏈重掃。  
2. 因為從未達 12 確認、未 `credited`：可用餘額不變；UI 從「確認中 6/12」變「未到帳／已失效」，可提示用戶用新 txHash 再查。若錯誤地已 credited：觸發異常工單，凍結出金，唔自動倒扣已花餘額。  
3. `credited` 只允許 `confirmations >= N`（或跟 `safe`／`finalized`）；scanner 可超前寫 `seen`，推進可用餘額要保守；冪等鍵 `chainId+txHash+logIndex` 防重掃雙加。
