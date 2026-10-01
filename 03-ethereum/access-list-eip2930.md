# Access List（EIP-2930）— CEX 預先報「會摸邊邊地址」

## 人話（從你熟的 CEX）

Type-1 起源嘅 **access list**：預先聲明呢筆 tx 會讀／寫邊啲地址同 storage slot。節點可以提早 warm 呢啲帳戶，**cold → warm** 嘅 gas 折扣用得上，estimate 同實扣更容易貼埋。

對 CEX 後端：出金／熱錢包 **transfer／sweep 歸集** 多數係固定路徑（同一 ERC-20、`transfer`／`transferFrom`、少數 router）。簽名前用靜態模板或 `eth_createAccessList` 產出 list，塞入 **type-2（1559）** tx 一齊簽——2930 定義 list 語意，1559 定義費用；實務上兩者共存於同一筆出金 tx。

## 面試短答

- EIP-2930 access list：可選字段，列地址（同可選 storage keys）。正確填寫令後續訪問按 **warm** 計費，減少冷帳戶突發成本同 estimate 偏差。
- CEX 場景：熱錢包 transfer／sweep、multisig、已知 bridge 路徑——由規則模板或 `eth_createAccessList`／tracer 生成，快取「鏈 × 合約版本 → list」。
- 同 type-2：**現代出金用 type-2 + optional `accessList`**；唔填 list 一般仍可上鏈，只係慳唔到／estimate 可能鬆一截。
- 唔係萬能：亂填無用地址令 calldata 變大；slot 估錯通常唔 revert，只係冇折扣。合約升級／proxy implementation 變要失效快取。

## 常見追問（連答案）

**Q: Access list 同 EIP-1559（type-2）點共存？**  
A: 2930 定義 access list 語意（最初同 type-1）；London 之後慣例係 **type-2 交易亦可帶 `accessList` 字段**。面試一句：費用市場跟 1559（maxFee／maxPriorityFee），warm 路徑跟 access list；CEX 熱錢包出金兩者一齊用，唔係二揀一。

**Q: `eth_createAccessList` 同手寫 slot map 點揀？**  
A: 高頻固定路徑（同一 USDT `transfer`）可快取模板（token 合約 + `balances[from/to]` 等 slot）；路徑一多或代理合約易變，優先 **`eth_createAccessList`（節點支援時）或 debug tracer** 對「即將廣播嘅 calldata」跑一次，再審核後快取。手寫 `keccak(abi.encode(key, slot))` 要跟緊 storage layout，proxy 升級係失效條件。

**Q: 唔填 access list 會唔會上唔到鏈？Sweep 一定要開？**  
A: 一般**唔會**上唔到。Access list 係 gas 優化（同少數規則下嘅明確聲明），唔係護照。CEX 可以新鏈先不上 list 保兼容；對日頻 sweep／大額出金再開啟，用「estimate 誤差／實際 gas」指標決定值唔值得維護模板。

**Q: 同預先 `eth_call`／`eth_estimateGas` 點分工？**  
A: 模擬解決「會唔會 revert／邏輯對唔對」；access list 解決「執行時 gas 會計（cold/warm）」。流程：simulate 過關 →（可選）`eth_createAccessList` → 帶 list 再 estimate／簽真 tx，減少 out-of-gas 同「估 6 萬實跑 8 萬」類事故。

## 小練習

**題：** CEX 熱錢包每日對同一 USDT 合約做大量 `transfer` 歸集（sweep），偶發換成另一套 proxy implementation。你會點設計 access list 快取同失效？（兩三句）

**參考答案：**  
按「鏈 × 代幣合約地址 × implementation／bytecode hash」快取模板 list（至少 token 合約地址；再加 hot wallet 與常用歸集目標相關 `balances` slots）。每次簽名前可用 cheap 檢查（implementation slot／codehash）核對版本；唔同就呼叫 `eth_createAccessList` 或 tracer 重生成並覆蓋快取。Sweep job 監控「有 list vs 無 list」嘅 gas 分位，proxy 升級後若漏失效，只係慳少咗，唔應當成交易失敗——但要告警避免長期用錯模板。
