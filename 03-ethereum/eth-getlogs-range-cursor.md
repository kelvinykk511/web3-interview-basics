# eth_getLogs 範圍限制與充值游標（CEX Indexer）

## 人話（從你熟的 CEX）

你當 Kafka consumer：要有 **offset／水位**，要能重放，範圍一次唔好拉爆 broker。鏈上 `eth_getLogs` 一樣——provider 會限 block 窗口、結果集大小；你要自己維護「掃到邊個 block」、reorg 時點回退。

`eth_subscribe("logs")` 適合跟 tip；歷史補洞、斷線重連、多合約回填，幾乎一定要 **`eth_getLogs` + 游標**。兩者係互補，唔係二選一。

## 面試短答

- **`eth_getLogs`**：按 `fromBlock`/`toBlock` + `address`/`topics` 查歷史／近端 logs；回傳含 `blockNumber`、`transactionHash`、`logIndex`、`removed`（部分實現）。
- **Provider 限額**：常見拒絕「範圍太大」（例如一次 >2k～10k blocks）、或結果集過大 → 要 **切段掃描**（固定步長或自適應縮小窗口）。
- **游標**：持久化 `lastProcessedBlock`（或 per-contract 水位）；下一輪 `from = cursor+1`，`to = min(head-confirmations, cursor+step)`。
- **確認數**：業務入帳水位 ≤ `head - N`；indexer 可超前寫 `seen`，最終態另標。
- **Reorg**：發現共同祖先 < 游標 → 回滾 DB 中未最終／被 `removed` 嘅 logs，游標後退，再前進（同消息隊列 rewind）。
- **冪等**：寫入鍵 `chainId + txHash + logIndex`（你投毒題用過），重掃唔雙花。

## 常見追問（連答案）

**Q：點解唔從 0 掃到 latest 一次過？**  
A：RPC 會 timeout／rate-limit；archive 壓力大；首次全量要分批 backfill。生產用「歷史 batch 作業 + 近端小窗口追 tip」。

**Q：`subscribe` 咗仲需唔需要 getLogs？**  
A：需要。WS 斷線會漏；進程重啟要從游標補；多實例切主要對齊水位。實務：subscribe 加速近端，getLogs 做可靠底盤。

**Q：步長固定 2000 會有咩問題？**  
A：熱門合約短窗口都可能結果爆炸；冷合約長窗口先有效率。可自適應：遇 provider 錯誤就對半切；空結果可加大步長。多 address 用 batch（注意 topics 係 OR／AND 語義）。

**Q：`removed: true` 係咩？**  
A：reorg 後原先 log 失效嘅通知（視客戶端／訂閱）。處理：對應冪等鍵作廢或標記，唔可以當「再入帳」。游標模型要同確認數策略一致，避免已 credited 先靠 removed 先知——所以最終入帳要等足夠深度。

**Q：同 The Graph／自建索引題點分工？**  
A：呢題講 **RPC 取 log 嘅工程細節**；自建 vs Graph 講產品選型。CEX 自建充值路徑幾乎一定要熟 getLogs 游標。

## 小練習

**題：** 游標停在 block 1_000_000，chain head=1_000_180，產品 12 確認，USDT 同 USDC 兩個合約。Provider 單次 getLogs 最多 2_000 blocks 且結果 ≤10k logs。你點設計下一輪請求？若掃到一半進程崩潰，重啟後點保證唔漏唔雙入？

**參考答案：**  
1. 業務可處理上界 = `1_000_180 - 12 = 1_000_168`。下一窗例如 `from=1_000_001`，`to=min(1_000_168, 1_000_001+step-1)`；兩個合約可分開請求或 `address=[usdt,usdc]`（確認 provider 語義）。  
2. 每成功處理完一個窗口（解碼 + 冪等 upsert + 標記 seen）先提交游標到 `to`；崩潰後從 DB 游標續跑。入帳 credited 另用確認數／最終性狀態機，重掃靠 `txHash+logIndex` 冪等。遇 range／結果過大錯誤：縮小 step 重試同一 `from`，切勿跳窗。
