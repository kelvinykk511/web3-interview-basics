# Stuck Nonce 恢復（CEX 出金運維 · 輕刷）

## 人話（從你熟的 CEX）

熱錢包出金似 **有序 outbox／消息隊列**：每個地址嘅 nonce 必須 `0,1,2,…` 連續消費。中間一筆 pending 死咗（Gas 低、被 mempool 丟、或廣播失敗），後面全部堵死——同「offset 卡住、後面 offset 消費唔到」一樣。

你要做嘅唔係「再開一條消息」，而係對 **同一個 offset（nonce）** 加速或取消：

- **加速（RBF）**：同 nonce、更高 fee，頂掉舊 pending
- **Cancel**：同 nonce、`to=自己`、`value=0`、更高 fee，清隊
- **絕對禁止**：同一筆出金業務單再開新 nonce——舊單之後突然上鏈 = **雙重出金**

類比：出金單 = 業務消息；nonce = 該地址嘅消費 offset；txHash 會隨 RBF 變，但業務綁定嘅係 `(from, nonce)`，唔係永遠綁死第一個 hash。

## 面試短答

- **偵測**：`eth_getTransactionCount(addr, "latest")` = 鏈上已確認嘅下一 nonce；`"pending"` = 含本地／節點視為 pending 嘅計數。`pending > latest` 且對應 hash 長期無 receipt → stuck 候選。
- **恢復優先序**：① 同 nonce **RBF 加速**；② 同 nonce **cancel**；③ 確認舊 tx 已被取代／丟棄／上鏈後先再前進下一筆。
- **Nonce gap**：誤發更高 nonce 而中間未上鏈 → 後面卡到缺口填上；要補發中間 nonce 或等驅逐／取消策略，唔好當「跳號就快啲」。
- **CEX 狀態機**：出金單綁 `(from, nonce)`；RBF 只更新 `txHash`／fee；業務層禁止「新 nonce 再出同一筆」。
- **串行分配**：熱錢包 nonce 用 DB 行鎖／單隊列 allocator；多 signer／多 worker 搶 nonce = gap 或雙花風險。
- **Evicted vs 仍 stuck**：`eth_getTransactionByHash` 變 `null` 且無 receipt ≠ 一定安全重開——可能只係你連嘅節點睇唔到，別的 mempool 仲有；要多節點／explorer 交叉，再决定釋放定重排隊。

## 常見追問（連答案）

**Q：`latest` 同 `pending` 差幾代表咩？**  
A：通常表示有若干筆仍被視為 pending（計上差異）。要逐筆對 hash：低費卡住、已被取代、節點已驅逐、定係已上鏈但 RPC 滯後。唔好只睇數字就開新 nonce。

**Q：點解永遠唔好為同一出金再開新 nonce？**  
A：同一 nonce 最終只會有一筆進塊；舊 pending 之後仍可能被打包。新 nonce 再出同一業務單 = 兩筆都可能上鏈 → 雙重出金。正確係同 nonce 取代，或確認舊 tx **永久唔會上鏈** 先釋放／重排隊。

**Q：Cancel 成功之後，業務單何時釋放 vs 重新入隊？**  
A：Cancel receipt `status=1` 且 `latest` 已越過該 nonce → 鏈上該 nonce 已「空轉」消耗。若用戶仍要出金：開 **新業務輪次**（新單號）再用 **新 nonce** 廣播——呢個唔係「同一筆雙重廣播」，而係取消後正式重做。若產品決定拒絕／退回：釋放凍結額度，狀態 `cancelled`，唔好默默用下一 nonce 再打舊單。

**Q：多 signer／多 worker 點樣出事？**  
A：兩個進程同時讀 `pending`、各自簽 nonce=N → 後到覆蓋或產生 gap；或一人 cancel、另一人以為已釋放又簽 N+1 同一業務。解法：單點 **serial nonce allocator**（行鎖／租約）、出金 worker 互斥、RBF／cancel 亦走同一鎖。

**Q：點分辨「mempool 已驅逐」vs「仲卡住」？**  
A：單節點 `getTransactionByHash=null` 唔夠。交叉：多 RPC／explorer、睇 `latest` 有冇前進、同 nonce 有冇替代 hash、本地是否曾 RBF。若確認全網睇唔到且超過驅逐／TTL 策略窗口，先可按 runbook 釋放；否則只准同 nonce 再廣播（加速或 cancel），唔好跳號。

## 小練習

**題：** 出金單 W（nonce=10）低費卡住；你對 nonce=10 做 cancel，cancel receipt 已 `status=1`，`latest` 變 11。同事話「用戶仲等緊錢，直接用 nonce=11 再廣播同一筆 W」。同時另一個值班仔睇到舊 W 嘅原 txHash 喺某個公共 explorer 仍顯示 pending。你會點裁決？業務狀態點走？

**參考答案：**  
Cancel 已上鏈 → 該地址 nonce=10 已被空轉消耗，**原 W 對應嘅轉帳唔會再以 nonce=10 出金成功**（同一 nonce 已有一筆進塊）。同事用 nonce=11「再廣播同一筆 W」本身可以係「取消後重做」，但前提係：① 業務層把舊 W 標成 `cancelled`／解凍再開 **新單 W2**（或明確重排隊標記），冪等鍵唔好同舊 W 混；② 忽略「舊 hash 仲 pending」嘅假陽性——多半係 explorer／節點快取滯後，以 **你廣播 cancel 嘅 receipt + `latest`** 為準，必要時多 RPC 核對。若 cancel 其實未成功而誤以為成功就開 11，先要停手查清。正確運維：serial 鎖內更新狀態 → 新單新 nonce → 禁止唔改單號就當「同一筆續傳」。
