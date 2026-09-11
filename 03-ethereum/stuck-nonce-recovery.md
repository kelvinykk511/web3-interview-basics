# Stuck Nonce 恢復（CEX 出金運維）

## 人話

熱錢包地址 nonce 必須連續：`0,1,2,…`。若 nonce=5 嘅 tx 卡喺 mempool（Gas 低／被丟），後面 nonce=6、7 **永遠上唔到鏈**——成條出金隊列塞住。呢題係 mempool／RBF 嘅 **運維升級版**：點偵測、點清隊、點避免雙花。

類比：訊息隊列要按序消費；中間一條 pending 死咗，後面全部堵死。你要嘛加速／取消嗰條（同 offset），要嘛唔好跳號重投業務單。

## 面試短答

- 偵測：`eth_getTransactionCount(addr, "latest")` = 已上鏈下一 nonce；`"pending"` = 含 mempool。若 `pending > latest` 且對應 hash 一直無 receipt → stuck
- 恢復優先序：① 同 nonce **加速（RBF）** 提高 fee；② 同 nonce **cancel**（to=self, value=0）；③ 確認舊 tx 已被取代或丟棄後先再前進
- 缺口（nonce gap）：若誤發 nonce=8 而 5–7 未上鏈，8 會卡住直至缺口填上——要麼補發 5–7，要麼等過期／取消策略依客戶端
- CEX 狀態機：出金單綁定 `(from, nonce, txHash)`；重試只允許同 nonce 重簽；業務層禁止「新 nonce 再出同一筆」
- 多 signer／併發：熱錢包出金要 **串行分配 nonce**（DB 行鎖／隊列），否則搶 nonce 或產生 gap

## 常見追問＋答案

Q：`latest` 同 `pending` nonce 差 3 代表咩？  
A：通常有 3 筆（或計數差 3）仍喺 mempool／本地視為 pending。要逐筆對 hash：係低費卡住、已驅逐（getTransaction 變 null）、定係已上鏈但 RPC 滯後。

Q：加速咗好多次仍然 pending，仲有咩招？  
A：檢查是否 nonce 前面仲有更早 pending；檢查節點是否廣播到足夠 peers；換更高 `maxFee`／`priorityFee`；必要時 cancel 清隊後按業務規則重新排隊（新業務輪次用新 nonce，唔係同一筆雙重廣播）。

Q：點解「清空卡住」最危險嘅做法係再開新 nonce 重打同一出金？  
A：若舊 pending 之後突然上鏈，新 nonce 又一筆 → **雙重出金**。正確係同 nonce 取代，或確認舊 tx 永久唔會上鏈（cancel 成功／過期策略）先釋放業務單。

Q：EIP-1559 之下 cancel tx 點組？  
A：同 nonce；`to=自己`；`value=0`；`data` 空；`maxFeePerGas`／`maxPriorityFeePerGas` 高過被取代嗰筆（節點通常要求明顯更高）；簽名後 `sendRawTransaction`。

## 小練習

題：熱錢包 `latest=10`，`pending=12`。出金單 A 用咗 nonce=10，B 用 11。A 低費卡住，B 亦上唔到。值班同事想「對用戶 B 用 nonce=12 再廣播一次」。會發生咩？正確步驟？

參考答案：危險。若之後 A／舊 B 任一上鏈，再加 nonce=12 可能變成額外一筆出金；且 10 卡住時 12 亦未必能進（視客戶端／缺口規則）。正確：對 nonce=10 做 RBF 加速或 cancel → 確認 10 落地或被取代 → 再處理 11；全程出金單唔改綁定 nonce；禁止為同一業務單開新 nonce。併發上要鎖住 nonce 分配器，避免第三人再搶 12。
