# eth_estimateGas 失敗診斷（CEX 提現預檢）

## 人話（從你熟的 CEX）

提現／歸集廣播前，後端通常先「乾跑」一次：問節點「呢筆 call 大概食幾多 gas」。  
`eth_estimateGas` 本質近 `eth_call` 模擬——**成功回數字；失敗多數係合約 revert**（唔係真扣咗 gas）。  
CEX 要分清：係參數錯、餘額唔夠、黑名單、allowance 不足，定係節點／區塊狀態過舊——先決定 retry、改單定人工介入。

## 面試短答

- `eth_estimateGas`：用同 `eth_call` 類似嘅 `from`／`to`／`value`／`data`，喺指定 block tag 模擬執行，回傳 **gas 上限估測**（通常略高於實際 `gasUsed`）。
- **失敗 ≠ 網絡問題**：多數 JSON-RPC error 帶 revert data；要 decode 做業務錯誤碼（例如 `TRANSFER_FROM_FAILED`、自訂 error）。
- **成功估到 ≠ 上鏈必成**：mempool 排隊期間狀態會變（餘額、nonce、oracle、黑名單）；仍要以 receipt 為準。
- CEX 實務：估 gas → 加 buffer（例如 20% 或固定 cap）寫入 `gas`／`gasLimit` → 簽名 → `sendRawTransaction`；估唔到就 **唔簽名廣播**，避免卡 nonce 嘅失敗 tx。

## 常見追問（連答案）

**Q：estimateGas 同 eth_call 差喺邊？**  
A：都係模擬。`eth_call` 回執行返回值／revert；`eth_estimateGas` 回「跑完要幾多 gas」。估 gas 時節點會試 binary search／逐步提高 gas limit。業務上：要讀返回值用 call；要定 gasLimit 用 estimate。

**Q：點解有時本地 call 過、estimate 又 fail（或相反）？**  
A：常見原因：`from` 唔同（權限／餘額）、block tag 唔同（`latest` vs `pending`）、缺 `value`、或 estimate 用嘅 gas cap 太低導致假 OOG。CEX：預檢要用 **同條即將簽名嘅 tx 一樣嘅字段**。

**Q：revert reason 點解出嚟？**  
A：error 入面嘅 data：舊式 `Error(string)`（selector `0x08c379a0`）或 Solidity custom error。用 ABI decode；解唔開就記 raw hex + 對合約源碼／已知錯誤表。唔好只展示「execution reverted」畀客服。

**Q：OOG（out of gas）同 revert 點分？對 CEX 意味咩？**  
A：估 gas 階段「gas 唔夠跑完」多數係模擬上限問題，可提高再試；合約 `require`／`revert` 係業務失敗，再加 gas 都唔會過。上鏈後：receipt.status=0 要對 logs／trace 定性，**失敗 tx 仍食 gas、仍佔 nonce**。

## 小練習

**題：** 熱錢包提 USDT，estimateGas 失敗，error data 解出係 blacklist／paused。你會點設計狀態機？同「RPC timeout」點分路徑？

**參考答案：**  
業務失敗（blacklist／paused／balance）：標記提現單 `PRECHECK_FAILED`／可關閉或等解禁，**唔分配新 nonce、唔廣播**；告警給風控／上幣。  
RPC／節點 transient：重試另一 endpoint、換 block tag、限次 backoff；仍失敗先 `NODE_UNAVAILABLE` 暫停該鏈出金，唔好當成用戶錯。  
共用原則：預檢失敗唔寫鏈；只有 estimate 成功先進入簽名／廣播；上鏈後只信 receipt。
