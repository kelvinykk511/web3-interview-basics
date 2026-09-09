# Mempool 與 Replace-by-Fee（卡住提現）

## 人話

你廣播咗提現 tx，但遲遲未上鏈——多數係 **Gas 太低，喺 mempool（待打包池）排隊／被丟棄**。CEX 後端要懂：點樣偵測 pending、點樣用 **更高 Gas 同同一個 nonce** 取代（RBF／加速），同埋點解唔可以另開一個 nonce 再送同一筆。

類比：訂單進「待撮合隊列」；你可以撤單重下更高價，但 **同一業務單號（nonce）** 只能有一筆最終成交。

## 面試短答

- 廣播後：節點驗證簽名／nonce／餘額 → 進 mempool → miner/validator 選高費優先打包 → 上鏈出 receipt
- pending 卡死常見因：`gasPrice`／`maxFeePerGas` 低於市價、nonce 前面有洞、節點 mempool 驅逐
- 加速／取消：用 **相同 from + 相同 nonce**，更高手續費重簽並再 `sendRawTransaction`（取代舊 pending）
- 取消（cancel）：同一 nonce，`to=自己`、`value=0`、更高 fee，把原提現「頂掉」
- 禁止：同一提現再開新 nonce 再廣播一筆——可能導致 **雙重出金**

## 常見追問＋答案

Q：點解加速一定要同一個 nonce？  
A：鏈上對每個地址按 nonce 嚴格排序；同一 nonce 最終只會有一筆進塊。提高 fee 重放同一 nonce = 取代候選，唔係多一筆轉帳。

Q：已進塊仲可不可以 RBF？  
A：唔可以。上鏈後只能等確認／再發新 nonce 嘅新交易。若要「撤回」已上鏈提現，鏈上做唔到；只能業務層凍結／追討。

Q：EIP-1559 之下加速改邊啲欄？  
A：通常提高 `maxPriorityFeePerGas` 同足夠嘅 `maxFeePerGas`（要蓋過 baseFee + tip）。仍係同 nonce 重簽。

Q：CEX 提現狀態機點接？  
A：`SIGNED/BROADCAST` → 輪詢 `eth_getTransactionByHash`（pending vs null）+ receipt；超時則 RBF 加速或 cancel；成功以 receipt.status=1 + 確認數為準，並用 txHash 冪等，切忌另開 nonce 重試同一筆業務單。
