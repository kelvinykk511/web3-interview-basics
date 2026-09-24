# debug_traceCall（廣播前深度模擬）

## 人話（從你熟的 CEX）

`eth_call` 告訴你「模擬結果／會唔會 revert」；有時仲要睇 **內部呼叫路徑**——例如 router 會唔會多跳、會唔會寫唔預期 storage、gas 會唔會喺某一層爆。  
`debug_traceCall` = 對 **未上鏈** 嘅一筆 call／擬發送 tx，用同 `debug_traceTransaction` 類似嘅 tracer 跑一遍。人話：上線前用 debugger step into，而唔係等生產炸咗先 `traceTransaction`。

CEX：高風險出金路徑（聚合器、跨合約、新 token）預檢可以「estimateGas + eth_call +（抽樣）traceCall」；日常小額 transfer 未必每次都 trace（成本高）。

## 面試短答

- **輸入**：同 `eth_call` 類似嘅 `from/to/data/value/gas` + block tag；可加 tracer（常用 `callTracer`）。
- **輸出**：呼叫樹／錯誤位置／每次 call 嘅 gas；用於理解 **點解** 會 revert，唔只係 boolean。
- **分工**：
  - `eth_estimateGas`：定 `gasLimit` 緩衝。
  - `eth_call`：業務返回值／快速 revert。
  - `debug_traceCall`：內部路徑、哪一層 revert、gas 熱點。
- **限制**：要 debug API；重放成本高；`stateOverride` 能否併用視客戶端；**模擬成功仍 ≠ 廣播後必成功**（mempool 搶跑、狀態已變、nonce 衝突）。

## 常見追問（連答案）

**Q：每次出金都 traceCall 會唔會太重？**  
A：會。實務分級：白名單 ERC-20 直 `transfer` → call + estimate 即可；新 token／router／多跳 → 強制 traceCall 或離線 fork 模擬；失敗重試前再 trace。監控 debug RPC QPS，同業務 RPC 分開。

**Q：traceCall 成功但 broadcast 後 receipt=0，常見原因？**  
A：模擬用嘅 block／state 同打包時不同（餘額被其他 tx 用咗、oracle 價變、allowance 被撤）；gas tip 導致交易延遲期間狀態漂移；或模擬 `from` 權限同實際簽名者一致但 nonce／balance 已變。所以預檢通過後仍要短 TTL、鎖定餘額、必要時重新預檢再簽。

**Q：同 Tenderly／Anvil fork 點選？**  
A：節點 `debug_traceCall`：低依賴、貼合你們 RPC、適合自動化預檢。Fork 模擬：可連續多筆、改時間戳、快照對比，適合複雜事故同研發。CEX 熱路徑優先 RPC；戰爭室／新集成用 fork。

**Q：callTracer 樹上見到 `DELEGATECALL` 要小心咩？**  
A：邏輯喺「被委託嘅 code」，storage 卻喺「呼叫方」——代理錢包／proxy token／多簽模組常見。排查 revert 時要跟 `DELEGATECALL` 目標 implementation，唔好只睇 proxy 地址嘅源碼。

## 小練習

**題：** 新上架一個「有 fee-on-transfer」嘅山寨 ERC-20，熱錢包要支援提現。你會點設計預檢？點樣用 traceCall 發現「轉出金額 ≠ 收款方收到」？

**參考答案：**  
1. 預檢唔只 `eth_call` `transfer` 成功：用 `callTracer` 睇有冇額外內部 `transfer`／扣費；對比執行前後（或 call 後再 `balanceOf(recipient)` 另一筆 `eth_call`）確認 **實際到帳額**。  
2. 產品層：對 fee-on-transfer／rebasing 幣，提現單應以「用戶到帳額」或明確「扣費由用戶承擔」規則入帳；風控可拒絕非標準 token。  
3. `gas`／`estimate` 仍要做；traceCall 抽樣確認冇奇怪 `DELEGATECALL` 去未知地址。  
4. 上架 checklist：標準 ERC-20 測試套件 + 非標準標記 + 預檢策略綁定到幣種配置，而唔係全局一套。
