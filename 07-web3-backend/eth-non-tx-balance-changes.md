# 冇交易都會加錢：ETH 嘅「非交易入帳」

> 2026-11-04 warm-up。對帳發現充值地址餘額多咗，但掃 tx / logs / trace 都搵唔到來源？多數係呢幾種。

## 人話
CEX 掃充值通常睇：普通 tx 嘅 `to`、ERC-20 `Transfer` log、trace 入面嘅 internal call。
但 ETH 有幾種加錢方式**完全唔係一條「打俾你」嘅交易**，淨係掃 tx 會漏，對帳就會出現「莫名其妙多咗錢」。

## 幾種來源
1. **驗證者提款（Shanghai/Capella 之後）**
   - 每個 block 有 `withdrawals` 列表（`index`, `validatorIndex`, `address`, `amount` 以 Gwei 計）。
   - 系統層直接加餘額：冇 tx、冇 log、冇 EVM call，trace 都見唔到。
   - 用 `eth_getBlockByNumber(n, false)` 攞 `withdrawals` 欄位先掃到。
   - 有人可以將 validator 嘅 0x01 withdrawal credentials 設成你嘅充值地址，之後每幾日自動有獎勵流入。
2. **出塊收益（fee recipient）**
   - Block 嘅 `miner`/`feeRecipient` 收 priority fee（同 MEV builder 付款，後者通常係 block 最後一條普通 tx，可以掃到）。
   - Priority fee 部分係直接加餘額，冇 tx。
3. **SELFDESTRUCT 送 ETH**
   - 合約 selfdestruct 會將餘額強制轉去 target，**唔會執行 target 嘅 receive/fallback**。EIP-6780（Cancun）之後多數情況唔再刪 code，但轉餘額呢個行為仲喺度。
   - trace 會見到（callTracer 有 SELFDESTRUCT frame），但只掃 tx `to` 同 logs 就會漏。
4. **普通 internal transfer**（合約 CALL 帶 value）— 之前講過要 trace，呢度唔重複。

## 後台設計要點
- **餘額差異對帳兜底**：每個 block（或每 N block）對熱點地址做 `balance(n) − balance(n−1)`，同「已識別入帳 − 已識別支出 − gas」比較；有差就入 `unexplained_delta` 表，告警 + 人工。
- **掃塊時一齊讀 `withdrawals`**：同一個 block 請求就有，成本好低；匹配到充值地址就生成 `deposit_source = BEACON_WITHDRAWAL` 嘅記錄。
- **入帳政策要寫清楚**：
  - 驗證者提款／SELFDESTRUCT 呢類「非正常充值」建議 **人工審核後入帳**，唔走自動入帳流程（無 tx hash 可以俾用戶對、可能係誤操作，亦可能係洗錢路徑）。
  - 記錄冇 tx hash，唯一鍵要用 `(block_hash, withdrawal_index)` 或 `(tx_hash, trace_path)`。
- **Reorg**：withdrawals 屬於 block，block 被 reorg 就跟住無效；照用同一套確認數／finalized 邏輯。
- **熱錢包唔好做 fee recipient / withdrawal credential**：自己嘅質押、出塊收益用獨立地址，避免混入用戶資金帳。

## 面試短答
> 除咗 tx、log、internal call，ETH 仲有冇交易嘅入帳：block 入面嘅 validator withdrawals、fee recipient 收 priority fee、SELFDESTRUCT 強制轉帳。我哋掃塊時同時讀 `withdrawals`，trace 捉 SELFDESTRUCT，再用每 block 餘額差異做兜底對帳；呢類非標準來源唔自動入帳，走人工審核，唯一鍵用 block hash + withdrawal index。

## 常見追問（連答案）
- **Q：點解 trace 都見唔到 validator withdrawal？**
  A：因為佢唔係 EVM 執行，係共識層喺 block 處理完 tx 之後直接改執行層餘額，所以只會出現喺 block 結構嘅 `withdrawals` 欄位。
- **Q：SELFDESTRUCT 打錢俾充值合約（例如 CREATE2 收款合約），合約邏輯會唔會被觸發？**
  A：唔會，receive/fallback 唔執行。所以靠合約 event 嚟記帳嘅設計會漏，要靠 trace 或者餘額對帳。歸集（sweep）時要按實際餘額，唔好按帳面記錄。
- **Q：餘額差異對帳點避免誤報？**
  A：要扣返自己出嘅 gas（`gasUsed × effectiveGasPrice`，L2 仲要加 L1 data fee），同埋只喺 finalized／確認後嘅 block 做；用同一個 block number 讀餘額，避免 RPC 節點之間高度唔一致。

## 小練習（題目 + 參考答案）
**題目：** 某充值地址喺 block N 餘額比 N−1 多咗 0.032 ETH，tx 掃描、Transfer log、trace 全部冇記錄。寫出排查順序。

**參考答案：**
1. 讀 block N 嘅 `withdrawals`，睇有冇 `address` = 該地址（amount 係 Gwei，0.032 ETH = 32,000,000 Gwei，好似 validator 獎勵）。
2. 睇 block N 嘅 `miner`/feeRecipient 係唔係呢個地址。
3. 對 block N 所有 tx 跑 callTracer，搜 SELFDESTRUCT 同帶 value 嘅 internal call 以呢個地址為 target。
4. 確認讀餘額嘅兩個 RPC call 用咗同一個節點／同一個 block hash（排除 reorg 或節點唔同步）。
5. 搵到來源 → 寫入 `unexplained_delta` 對應記錄，轉人工審核決定入帳；搵唔到 → 升級告警。
