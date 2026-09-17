# On-chain Oracle（Chainlink 等）vs CEX 報價引擎

## 人話

CEX 入面，價格多數嚟自**撮合盤口、指數、內部標記價**——你信自己嘅撮合同風控。  
鏈上 DeFi（借貸清算、永續、穩定幣掛鉤）冇中央撮合，合約要問一個**鏈上可驗證嘅價格來源**：常見就係 **Oracle**（例如 Chainlink Aggregator：多節點報價 → 聚合 → 寫上鏈）。

對 CEX 工程師：你哋已經有「標記價／指數價／保險基金」思維；面試要講清楚——**鏈上合約信嘅係 oracle 回傳，唔係你交易所 API**。充提、Treasury 對沖、鏈上清算監控時，要分清「內部價」同「鏈上結算價」可以短暫脫鉤。

## 面試短答

Oracle 把鏈下／多源價格以可信方式寫入鏈上，合約用 `latestRoundData` 等讀價。  
Chainlink 典型模型：多独立 node 聚合、心跳＋偏差閾值更新、帶 `updatedAt`／roundId 防過期。  
CEX 對照：內部標記價服務 ≈ oracle 角色，但信任模型唔同——CEX 信自家系統；鏈上信 oracle 合約＋節點集。面試加一句：用價前要檢查**過期、小數位、answer 是否 > 0、proxy 地址係咪官方**。

## 常見追問（連答案）

**Q: 點解 DeFi 唔直接 call CEX REST API？**  
A: 合約執行要決定論、可驗證；HTTP API 唔喺 EVM 共識入面。Oracle 係「把外部數據帶入共識可讀狀態」嘅橋；你可以用自己做 keeper 寫價，但風險／信任要自己扛。

**Q: Oracle 過期或卡住會點？**  
A: 借貸可能暫停清算或錯誤清算；永續資金費／強平用舊價。防禦：讀 `updatedAt` 設 max staleness、二次來源、熔斷。CEX 類比：標記價源掛掉就切備用指數、停新開倉。

**Q: CEX 做鏈上產品（例如鏈上抵押借貸）點用價？**  
A: 產品結算／清算應用**鏈上 oracle**（同協議一致）；風控儀表板可以同時睇 CEX 標記價做預警。兩邊價差大時警報，唔好用內部價去「覆蓋」鏈上清算結果。

**Q: Push vs Pull oracle？**  
A: Push（如傳統 Chainlink feed）有人／網絡定期寫鏈；Pull／on-demand 由用家交易時帶簽名報價上鏈付 gas。CEX 後端若做 keeper，要懂更新頻率同 gas 成本。

## 小練習

**題：** 你監控某個借貸協議，發現 Chainlink ETH/USD 已 10 分鐘冇更新，同時 CEX 標記價急跌 8%。協議清算會點？你做 CEX 風控會點做？

**參考答案：**  
鏈上若仍用舊 oracle 價，清算可能**滯後或暫時唔觸發**（取決於協議 staleness 檢查：有啲會 revert 用價）。CEX 風控應：（1）當協議暫停／高風險，限制相關充提或對沖；（2）儀表分開顯示「CEX 標記價」vs「鏈上 last oracle」；（3）唔好假設鏈上已跟跌——直到新 round 上鏈或協議熔斷邏輯生效。
