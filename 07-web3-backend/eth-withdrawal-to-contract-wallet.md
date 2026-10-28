# ETH 出金到合約地址（Safe / AA 錢包）：gas、失敗、退款

> CEX 橋樑：你哋出金系統以前假設「收款方 = EOA（普通私鑰地址）」，轉 ETH 一定成功、21000 gas 搞掂。但而家好多用戶用 Safe 多簽、ERC-4337 智能錢包、EIP-7702 委託咗嘅 EOA 收錢——收款方係**會執行代碼**嘅地址。

## 人話
- 轉 ETH 去 EOA：唔執行代碼，固定 21000 gas，幾乎唔會失敗。
- 轉 ETH 去合約：會觸發對方 `receive()` / `fallback()`，可能要多啲 gas，甚至**對方代碼主動 revert**。
- 交易失敗（receipt `status = 0`）都**照扣 gas**，而且錢冇轉出。你帳本要識得「已上鏈但失敗」呢個狀態。

## 面試短答
1. 出金前 `eth_getCode(to)`：非空 = 合約（或 7702 委託 EOA，code 以 `0xef0100` 開頭）。
2. 唔好寫死 21000：對合約地址用 `eth_estimateGas` + buffer（例如 ×1.2–1.5），設上限防對方吃爆 gas。
3. 如果出金係經自家合約批量發（batch payout），合約內用 `call{value:}` 而唔係 `transfer()`/`send()`——後兩者只俾 2300 gas，Safe proxy 等收款會失敗。
4. 用 `call` 就要防重入（checks-effects-interactions / ReentrancyGuard），而且批量入面**單筆失敗唔可以拖死成批**：記錄失敗再單獨處理（pull pattern / 失敗名單）。
5. 失敗交易：`status=0` → 出金單轉「鏈上失敗」，gas 成本入交易所費用帳，**用戶餘額解凍退回**或人工復核；唔好自動無限重試（對方 revert 係確定性嘅）。

## 常見追問（連答案）
**Q1：點解 `transfer()` 2300 gas 以前夠、而家唔夠？**
A：2300 係俾收款方 emit 一個 event 嘅「零用錢」，唔夠寫 storage。Safe 係 proxy，`receive` 要 `delegatecall` 去 singleton；EIP-2929（Berlin）之後冷帳戶/冷 storage 訪問變貴，所以好多合約錢包喺 2300 內做唔完 → revert。業界建議用 `call`。

**Q2：estimateGas 成功，上鏈都可能失敗？**
A：會。estimate 係喺「當前狀態」模擬；上鏈時狀態可能變咗（對方合約升級、暫停收款、狀態依賴）。所以要讀 receipt `status`，唔可以「tx 已打包 = 成功」。

**Q3：用戶話「錢冇到」，但 receipt status=1？**
A：去 Safe 嘅話會有 `SafeReceived` event；ETH 經合約入帳**唔會有 ERC-20 Transfer log**。用戶個 indexer/錢包 UI 可能只睇 log 或者只睇 external tx。俾 tx hash + `debug_traceTransaction`（callTracer）證明 value 去咗邊。

**Q4：出金地址係某條鏈嘅 Safe，用戶揀錯鏈點算？**
A：Safe 地址係 CREATE2 算出嚟，**同一地址喺另一條鏈可能未部署或屬於其他人**。轉 ETH 去未部署地址會成功但資金卡住，要等用戶喺嗰條鏈用相同參數部署先拎得返（未必可行）。所以出金頁對合約地址要顯示警告/要求白名單確認（地址+鏈）。

**Q5：對 7702 委託嘅 EOA 出金有咩分別？**
A：code 係 `0xef0100 || delegate 地址`，收 ETH 會跑 delegate 合約邏輯，可能 revert 或耗更多 gas。處理方式同合約錢包一樣：estimate + 讀 status。

## 小練習（題目 + 參考答案）
**題：** 你哋 batch payout 合約一次發 100 筆 ETH，用 `for` loop + `payable(to).transfer(amt)`。上線後第 37 個用戶係 Safe，成批 revert，其他 99 個都收唔到。點改？

**參考答案：**
1. 改用 `(bool ok, ) = to.call{value: amt, gas: CAP}("")`，CAP 例如 50k，防單一收款方吃光 gas。
2. `ok == false` 唔 revert 成批，emit `PayoutFailed(id, to, amt)`；後端監聽呢個 event 把該出金單轉「失敗待處理」並退回用戶可用餘額。
3. 加 `nonReentrant`，或者先標記已付再 call（CEI）。
4. 後端以 event + receipt 對帳：成功筆數 + 失敗筆數 = 100，金額對得上合約餘額變化。
5. 長遠：對 `eth_getCode` 非空嘅地址走單筆出金通道（獨立 estimate），唔混入批量。
