# Access List（EIP-2930）— CEX 預先報「會摸邊邊地址」

## 人話

Type-1 交易可以帶一份 **access list**：預先聲明這筆 tx 會讀／寫邊啲地址同 storage slot。  
好處係：節點可以提早 warm 呢啲帳戶，**cold → warm** 嘅 gas 折扣可以用上；同埋某啲情況下可以避免「未聲明就摸到」嘅額外昂貴 gas。

對 CEX 後端嚟講：出金／歸集多數係**固定合約 + 固定路徑**（例如某 ERC-20 嘅 `transfer`、某 bridge contract）。你可以喺簽名前由 indexer／ABI 推導出會碰到嘅 `to`、token 合約、必要 slot，塞入 access list，令 gas 估價更穩、實際花費更貼近模擬。

## 面試短答

EIP-2930 access list 係可選字段，列出地址（同可選 storage keys）。正確填寫可令後續訪問按 warm 計費，降低不可預期嘅 cold-account 成本。  
CEX 應用場景：已知合約互動（ERC-20／multisig／bridge）時，由模擬（`eth_call`／`eth_estimateGas` + tracer）或靜態規則生成 access list，再帶入簽名廣播。  
唔係萬能：亂填無用地址唔會幫你慳，仲可能令 tx 變大；slot 估錯亦唔等於執行失敗，只係慳唔到應有折扣。

## 常見追問（連答案）

**Q: Access list 同 EIP-1559（type-2）點共存？**  
A: 現代常用 **type-2（1559）交易亦可帶 accessList 字段**（Berlin／London 之後慣例）。面試可答：2930 定義 list 語意；1559 定義費用市場；實務上出金服務用 type-2 + optional accessList。

**Q: 點知要填邊啲 storage key？**  
A: 對 ERC-20 `transfer`，常見係 `balances[from]`／`balances[to]` 對應 slot（視合約 layout：簡單 mapping 可用 `keccak(abi.encode(key, slot))`）。更穩陣做法係跑一次 debug tracer／`eth_createAccessList`（若節點支援）自動產出，再快取「合約版本 → list 模板」。

**Q: 唔填 access list 會唔會上唔到鏈？**  
A: 一般唔會。Access list 係優化同（少數協議規則下的）明確聲明，唔係「必填護照」。CEX 可以先不上 list 保兼容，再對高頻路徑逐步開啟以慳 gas／穩定 estimate。

**Q: 同「預先 `eth_call` 模擬」有咩關係？**  
A: 模擬解決「會唔會 revert／邏輯對唔對」；access list 解決「執行時 gas 會計點樣」。兩者一齊用：先 simulate 過關，再帶 list 簽真 tx，減少 estimate 同實際 gas 偏差導致嘅 out-of-gas。

## 小練習

**題：** CEX 熱錢包每日對同一 USDT 合約做大量 `transfer` 歸集。你會點設計 access list 快取？（兩句）

**參考答案：**  
按「鏈 × 代幣合約地址 × 合約 bytecode/version」快取模板 list（至少包含 token 合約地址；有需要再加 hot wallet 同常用歸集目標相關 slots）。合約升級／proxy implementation 變更時使快取失效，並用 `eth_createAccessList` 或 tracer 重生成，避免用過期 slot map。
