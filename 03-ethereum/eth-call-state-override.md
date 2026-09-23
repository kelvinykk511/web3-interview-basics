# eth_call + stateOverride（出金／歸集前模擬）

## 人話（從你熟的 CEX）

`eth_estimateGas` 答「大概食幾多 gas」；業務有時仲要問「跑完會回咩、會唔會 revert、假設餘額／storage 係某狀態又點」。  
`eth_call` 喺指定 block 模擬執行，**唔改鏈、唔花 gas**；進階參數 **stateOverride** 可以臨時改某個地址嘅 balance／nonce／code／storage，再跑同一筆 call——似單元測試 mock DB，但對象係 EVM 狀態。

CEX 用途：提現預檢（唔止估 gas）、排查「用戶以為有幣但其實 allowance／掛起訂單鎖住」、模擬「若先 sweep 再 withdraw 會唔會過」、讀 view 時強制某 storage slot。

## 面試短答

- `eth_call`：`from`／`to`／`value`／`data` + block tag（`latest`／`pending`／具體高度），回傳執行結果或 revert data；**不上鏈**。
- **stateOverride**（多數節點支援，字段名因客戶端略異）：對一組地址覆蓋 `balance`、`nonce`、`code`、`state`／`stateDiff`（storage slot）。模擬完即棄，唔影響真實狀態。
- 同 `eth_estimateGas`：都係模擬；call 重結果／revert，estimate 重 gas。生產預檢常 **兩步都做**：call／decode 業務結果 + estimate 定 gasLimit。
- CEX：預檢要用 **即將簽名嘅同一組字段**；override 只用於「假設性分析／工具排查」，**唔好拿 override 成功當成鏈上必成**——真實 broadcast 冇呢層 mock。

## 常見追問（連答案）

**Q：點解有 estimateGas 仲要 eth_call？**  
A：estimate 失敗多只話 reverted；call 可以攞返回值（例如 router quote）或 decode 精確 custom error。有啲路徑 gas 估到但業務上要拒絕（回傳碼表示 partial fill／slippage）——要靠 call 讀返回值。

**Q：stateOverride 改 balance 之後 call 成功，可唔可以當作用戶有足夠資金？**  
A：唔可以。嗰次成功只證明「喺假狀態下 bytecode 邏輯過」。真實提現仍要對 **鏈上真實餘額 + 本系統帳本鎖額 + nonce**。Override 係 debug／設計驗證工具，唔係風控依據。

**Q：同 Tenderly／Anvil fork 模擬差喺邊？**  
A：`eth_call`+override 輕量、貼住你哋 RPC、適合線上預檢路徑。Fork 模擬（本地 anvil／第三方）可以跑多筆連續 tx、改時間、睇 trace，適合複雜排查，但延遲同依賴更高。CEX 熱路徑通常：RPC call／estimate；離線／工單再用 fork+trace。

**Q：block tag 用 latest 定 pending？**  
A：讀狀態／對帳多用 `latest`（或 `safe`／`finalized` 視鏈）。要估「連 mempool 未上鏈 tx」影響先考慮 `pending`——但節點對 pending 語義不一致，CEX 出金預檢多數鎖 `latest` + 自己維護 in-flight nonce／鎖定餘額，避免雙花同假成功。

## 小練習

**題：** 熱錢包要轉 USDT 去用戶提現地址。你想預檢：假設 storage 入面 token 合約對熱錢包嘅 balance slot 係 X（或先 mock ETH 夠付 gas），call `transfer` 會唔會 revert。點用 stateOverride？成功後仲要做咩先敢簽名？

**參考答案：**  
1. 組同正式 tx 一樣嘅 `from=hot`、`to=USDT`、`data=transfer(to,amount)`。  
2. stateOverride：必要時提高 `hot` 嘅 ETH `balance`（付 gas 唔影響 ERC-20 邏輯，但有啲預檢腳本會一併設）；若要測「餘額剛夠／剛唔夠」，用 `stateDiff` 寫 USDT 合約裡該地址 balance 對應 slot（要知 storage layout；唔知就唔好瞎改，改用真實 chain 狀態 + 細額測試地址）。  
3. `eth_call` 成功／revert decode → 再 `eth_estimateGas` → 先對 **真實** `balanceOf(hot)` 同內部鎖額校驗 → 先簽名廣播。Override 成功永遠唔跳過真實餘額同 receipt 確認。
