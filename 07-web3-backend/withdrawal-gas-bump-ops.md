# 提現 Gas Bump 實務（RBF + eth_feeHistory）

## 人話（從你熟的 CEX）

熱錢包出金廣播咗，用戶睇住「處理中」，鏈上一直無確認——多數係 tip／maxFee 跟唔上擁塞。你已經識 mempool／RBF（同 nonce 取代）同 fee oracle 概念；呢題係 **值班 playbook**：幾時 bump、bump 幾多、點用 `eth_feeHistory` 動態 tip、點同提現狀態機對齊，同埋點避免「好心再開一筆」變成雙重出金。

類比：撮合撤單重下更高價——訂單號（nonce）不變，只改報價；唔好另開一張單再賣一次。

## 面試短答

- **觸發**：`BROADCAST/PENDING` 超過 SLA（例如 N 個區塊／M 分鐘）仍無 receipt，或 `maxFee < 當前 baseFee + tip` → 進入 bump。
- **動作**：同一 `from` + 同一 **nonce**，重簽更高 `maxPriorityFeePerGas`／`maxFeePerGas`（EIP-1559 type=2），再 `sendRawTransaction`；狀態機把業務單綁到 **新 txHash**，舊 hash 標 `replaced`。
- **定價**：用 `eth_feeHistory(blockCount, newest, rewardPercentiles)` 睇近期 baseFee 同區塊 tip 分位（例如 25/50/75）；目標 tip 取 urgency 檔（普通／加急），`maxFee ≈ estimatedNextBaseFee × buffer + tip`。
- **升級路徑**：① bump tip／maxFee → ② 仍卡則再 bump（有上限／成本熔斷）→ ③ 同 nonce **cancel**（`to=self, value=0`）清隊 → ④ 確認舊候選唔會上鏈後，先按業務規則決定重排隊（新業務輪次）定人工。
- **硬規則**：禁止為同一 `withdrawId` 開 **新 nonce** 再廣播；nonce 分配要串行；每次 bump 記審計（舊 hash、新 hash、fee、原因）。

## 常見追問（連答案）

**Q：`eth_feeHistory` 回咩？點拆成 tip？**  
A：回每個歷史塊嘅 `baseFeePerGas`、同可選嘅 `reward`（該塊內交易實際 tip 嘅百分位）。出金 oracle 通常用最近 N 塊：baseFee 用短 EMA／最後一塊×緩衝估下一塊；tip 用目標百分位（擁塞揀更高，例如 p75）夾一個地板／天花板。唔好只用「上一次成功出金嘅 fee」當永恆真理。

**Q：bump 幅度點定？一次加 10% 夠唔夠？**  
A：節點／客戶端對置換通常要求 **明顯更高** 嘅 fee（常見經驗係 priority／maxFee 要比被取代嗰筆高一截，具體以你連嘅節點同鏈為準）。實務會用「相對舊 tx 乘子」（如 1.1–1.25×）同「相對當前 feeHistory 目標取 max」兩條取較大，避免「加咗但仍低於市價」。設成本上限，防止擁塞時無限抬價燒穿 hot wallet ETH。

**Q：pending 變 `null`（getTransactionByHash）但未有 receipt，算唔算可以開新 nonce？**  
A：**唔等於**舊 tx 永遠唔會上鏈——可能只係你嘅節點 mempool 丟咗，別的 relay 仲有。正確：再廣播同 nonce 嘅高費版／cancel，或多節點交叉查；只有 cancel 上鏈、或過了你定義嘅「確定丟棄」策略並完成對帳後，先釋放該 nonce／業務單。否則雙重出金風險仍在。

**Q：同「未知 pending＝安全事件」（熱錢包監控）點分工？**  
A：Gas bump 處理 **我哋自己簽過、狀態機認識** 嘅出金 hash。若 `pending` nonce 窗口出現 **無對應出金單** 嘅 hash → 先當安全事件（暫停出金、查 key／流程），唔好當普通低費去 bump 別人嘅 tx。

## 小練習

**題：** 出金單 W1：`nonce=42`，`maxFee=30 gwei`，`tip=1 gwei`，已廣播 hash=H1。8 分鐘後仍無 receipt；`eth_feeHistory` 顯示近塊 baseFee≈28 gwei，p50 tip≈2 gwei，p75≈5 gwei。值班想：(a) 用 nonce=43 再打一筆相同金額給用戶；(b) 同 nonce 把 tip 改 1.1 gwei、maxFee 改 31 gwei。邊個錯？你會點做？

**參考答案：**  
(a) **嚴禁**——若 H1 之後上鏈，再加 nonce=43＝雙重出金。  
(b) 幅度可能仍不夠：`maxFee=31` 只勉強蓋住 baseFee，擁塞再升就再次失效；tip 1.1 亦低於市價分位。  
正確：同 nonce=42 重簽，tip 對齊 urgency（至少 p50–p75，例如 5 gwei），`maxFee` 用估下一塊 baseFee×buffer（如 1.25–2×）+ tip；廣播得 H2，W1 綁 H2、H1=`replaced`；繼續盯 receipt；若連續 bump 觸成本熔斷 → cancel 清隊並告警，而唔係開新 nonce。
