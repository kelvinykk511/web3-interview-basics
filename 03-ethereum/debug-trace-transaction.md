# debug_traceTransaction／callTracer（出金失敗排查）

## 人話（從你熟的 CEX）

用戶提現 `status=0`（receipt 失敗），運營只見到「reverted」，唔知係哪個內部 `transfer`、哪個 router hop、定係自訂錯誤。  
CEX 後端似睇 Java 堆棧：`debug_traceTransaction` 重放該筆已上鏈 tx，回傳呼叫樹（誰 call 誰、value、gas、output／error）。常用 **callTracer** 攞樹狀 internal calls——排查「熱錢包簽出去、合約中間 revert」必備。

注意：唔係所有公共 RPC 都開 `debug_*`；生產多用自建／付費節點，或 Tenderly 等模擬服務。面試要講清「能力」同「唔好當每個 endpoint 都有」。

## 面試短答

- **用途**：對 **已上鏈** `txHash` 重放執行，睇 internal call／CREATE、每次 call 嘅 `error`／`revertReason`、gas 用盡位置。
- **常用 tracer**：
  - `callTracer`：呼叫樹（from/to/type/value/input/output/error/calls[]）；`withLog` 可帶 log。
  - `prestateTracer`：執行前涉及地址嘅 balance／nonce／code／storage（分析「邊個 storage 被讀寫」）。
  - 結構化 tracer（client 異）：有啲用 `tracerConfig` 開 `onlyTopCall` 減 payload。
- **同 receipt**：receipt 畀 `status`、頂層 logs、gasUsed；**唔**展開內部 call。Trace 補「點解 status=0」。
- **CEX 流程**：出金 SM 見 `FAILED` → 拉 receipt → 若 `status=0` →（有權限）trace → decode revert → 工單分類（餘額不足／pause／黑名單／slippage／自訂 error）→ 決定重試／換路由／人工。

## 常見追問（連答案）

**Q：trace 會唔會改鏈上狀態？**  
A：唔會。係節點用歷史 state 重放該 tx，只讀分析。但重放貴 CPU，對公開節點常限流或關閉；CEX 應走專用 debug RPC 池，同用戶讀寫流量隔離。

**Q：點解有時 receipt 失敗但 trace 頂層 error 係空？**  
A：可能 gas 耗盡（out of gas）、低層 client 版本唔填 `revertReason`、或只係 `INVALID`／panic。要睇 call 樹上最深失敗節點嘅 `error`／`output`（revert data），再用 ABI decode。唔好只信頂層一句。

**Q：同 eth_call 失敗資訊差喺邊？**  
A：`eth_call`／`estimateGas` 係 **未上鏈** 模擬，適合預檢。`debug_traceTransaction` 係對 **已發生** 嘅真實 tx 做屍檢。預檢用 call／estimate；事故用 trace。兩者都應 decode 同一套 custom error。

**Q：L2 上 trace 稳唔稳？**  
A：多數 L2 有類似 debug API，但 tracer 名／字段、同「係咪完整 EVM 等價」要因鏈查。CEX 多鏈出金要按鏈維護「可唔可以 trace、用邊個 provider」矩陣，唔好假設 eth_call 得就一定有 callTracer。

## 小練習

**題：** 熱錢包出金 ERC-20，`txHash` 已上鏈，`receipt.status=0`，用戶投訴「錢扣咗未到」。你會按咩順序查？點向業務解釋「鏈上失敗 ≠ 帳本一定要退」？

**參考答案：**  
1. 對帳本：出金單狀態、是否已 `debit`、是否綁定該 `txHash`（防重複綁錯 hash）。  
2. `eth_getTransactionByHash`／receipt：確認 `from` 係熱錢包、`to` 係 token／router、`status=0`、`gasUsed` 是否貼 `gasLimit`（疑 out of gas）。  
3. `debug_traceTransaction` + `callTracer`：搵最深 revert 節點，hex `output` → ABI decode（例如 `Error(string)`／`InsufficientBalance`／`EnforcedPause`）。  
4. 帳務：鏈上失敗通常應 **解凍／退回可用餘額**（視你們 SM：`BROADCAST`→`ONCHAIN_FAILED`→`REFUNDED`），但要冪等——同一單唔因重掃 receipt 退兩次；若其實係「到錯合約但 status=1」則另案。向業務講：用戶所見「扣款」多係系統鎖額；真正鏈上轉帳失敗就要按 SM 退鎖，並用 revert 原因決定可否自動重試。
