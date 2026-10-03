# debug_trace／callTracer 輕刷（支援／風控排查）

## 人話（從你熟的 CEX）

用戶／風控：「鏈上失敗咗，錢去咗邊？邊層合約 revert？」Receipt 只俾你 `status=0` 同頂層 logs——好似 HTTP 500 冇 stack trace。  
`debug_traceTransaction` + **callTracer** = 對**已上鏈** tx 做屍檢：內部 call 樹（誰 call 誰、value、input／output、error）。  
日常預檢用 `eth_call`／`estimateGas`；事故同怪 tx 先上 debug。唔係每個公共 RPC 都開 `debug_*`——生產用自建／付費 debug 池，同業務讀寫流量隔離。

類比：Java 生產 core dump／分布式 tracing——receipt 係「成功／失敗」；callTracer 係 span 樹，睇邊個下游拋錯。

## 面試短答

- **eth_call／estimateGas**：未上鏈模擬——預檢會唔會 revert、估 gas。快、常見、多數節點有。
- **debug_traceTransaction**：已發生 tx 重放——internal calls、最深 revert、gas 耗盡位置。要 debug API；CPU 重。
- **callTracer**：樹狀 `from/to/type/value/input/output/error/calls[]`；可 `withLog`；`onlyTopCall` 減 payload。
- **CEX triage**：出金／怪轉帳 → receipt →（status=0 或「成功但金額怪」）→ trace → ABI decode revert／跟 value 轉移 → 工單分類（餘額／pause／黑名單／slippage／非標準 token）→ SM 退鎖或重試策略。
- **唔會改鏈上狀態**；L2 有類似 API 但字段／可用性按鏈維護矩陣。

## 常見追問（連答案）

**Q：支援單「reverted」——你會 eth_call 重放定 debug_trace？**  
A：已有 `txHash` 且已上鏈 → **優先 `debug_traceTransaction`**，因為要重放**當時**真實執行路徑（含嗰刻 storage／balance）。`eth_call` 用「而家」嘅 latest／某個 block 模擬，狀態可能已變，解釋唔到「當日點解爆」。預檢下一筆重試先用 call／estimate；屍檢用 trace。

**Q：receipt.status=1 仲要唔要 trace？**  
A：要——當業務對唔上時：例如「用戶話冇到帳但 status=1」、fee-on-transfer、路由器多跳只到中間合約、internal ETH 轉走。callTracer 跟 `value`／ERC-20 `transfer` 內部 call，對齊 indexer 同帳本。status=1 ≠ 產品意義上嘅「出金成功到用戶地址」。

**Q：頂層 error 空白、但 status=0？**  
A：常見 out of gas（`gasUsed` 貼 `gasLimit`）、client 未填 `revertReason`、或低層 panic。要落 call 樹最深失敗節點睇 `error`／`output`（revert data）再 ABI decode（`Error(string)`／custom error）。唔好只信頂層一句。

**Q：同 debug_traceCall 點分工？**  
A：`traceCall`＝廣播**前**對擬發送 tx 做帶呼叫樹嘅模擬（高風險路徑／新 token 抽樣）。`traceTransaction`＝廣播**後**屍檢。熱路徑：白名單直 `transfer` → call + estimate；事故／爭議 → traceTransaction。兩者都要 decode 同一套 error ABI。

**Q：風控點用 trace，而唔只係「失敗就退款」？**  
A：分類决定動作：可重試（臨時擁塞／slippage）vs 拒絕資產（pause／黑名單／非標準 token）vs 安全事件（to 地址唔係預期 router、怪 `DELEGATECALL`）。盲退款唔教系統；trace 原因碼寫入工單同監控，先可以自動重試或凍結幣種。

## 小練習

**題：** 熱錢包經 aggregator 出金，`receipt.status=0`，用戶投訴「餘額鎖咗」。支援想用 `eth_call` 復现同一筆 calldata 喺 latest；風控想立刻解凍。你點排？

**參考答案：**  
1. 對帳：出金單、`txHash`、是否已 debit／鎖額（防錯綁 hash）。  
2. Receipt：`from`／`to`／`gasUsed` vs `gasLimit`、status=0。  
3. **`debug_traceTransaction` + callTracer**（唔好只靠 latest `eth_call`）→ 最深 revert → decode（如 `STF`／slippage／`TransferFromFailed`）。  
4. 帳務：按 SM `ONCHAIN_FAILED` → 冪等解凍／退可用；同一單唔因重掃退兩次。  
5. 決策：可重試（調整參數／降 ROUTER）vs 停用該路徑；用 latest `eth_call`／`traceCall` 只服務「下一筆會唔會過」，唔代替屍檢結論。向業務講：鎖額係系統行為；鏈上失敗要退鎖，原因用 trace，唔好空口「區塊鏈壞咗」。
