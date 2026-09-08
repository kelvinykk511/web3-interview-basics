# eth_call vs eth_sendRawTransaction（讀鏈 vs 寫鏈）

## 人話

對 CEX 後端嚟講：  
- **eth_call** = 「模擬／唯讀查合約」——唔上鏈、唔花 Gas、唔改狀態（好似 `SELECT` / 預覽）。  
- **eth_sendRawTransaction** = 「廣播已簽名交易」——要上鏈、要 Gas、會改狀態（好似真正 `INSERT`／轉帳）。

提現、歸集要 sendRaw；查餘額、估 Gas、預跑合約邏輯用 call（或 `eth_getBalance`／`eth_estimateGas` 等讀接口）。

## 面試短答

- `eth_call`：指定 `to` + `data`（可選 `from`），喺某 block tag（`latest`／某個高度）執行，返回結果；**唔產生 txHash、唔改世界狀態**
- `eth_sendRawTransaction`：提交 RLP 編碼嘅已簽名 raw tx；節點驗證簽名／nonce／餘額後進入 mempool；之後先有 receipt
- 估 Gas：常用 `eth_estimateGas`（本質近 call 模擬）；真正扣費以鏈上執行為準
- 安全：call 結果唔等於上鏈成功；提現必須等 receipt + 確認數，唔能只信本地模擬

## 常見追問＋答案

Q：點解提現唔可以只做 eth_call？  
A：call 唔會真轉走資產。佢只告訴你「若果依家執行可能點」。真正出金一定要簽名並 `sendRawTransaction`。

Q：call 失敗係咪等於交易一定失敗？  
A：多數情況相關（revert 原因類似），但狀態會變（nonce、餘額、區塊條件）。生產仍要以 receipt.status 同業務對帳為準。

Q：CEX 歸集／提現簽名喺邊度做？  
A：私鑰唔應落應用機。常見：HSM／託管簽名服務簽好 raw tx，業務服務只負責組裝 unsigned tx + 廣播 + 追蹤 receipt。
