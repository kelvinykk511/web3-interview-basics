# eth_feeHistory Tip 選型（熱錢包／出金）

## 人話（從你熟的 CEX）

CEX 出金有「普通／加急」佇列——用戶付唔同提幣費，你就要唔同 **urgency**。鏈上對應唔係一個 `gasPrice`，而係 EIP-1559 兩截：

- **baseFee**：協議燒嘅，跟區塊擁塞自動升降——你**唔可以**靠加 tip「抵銷」過低嘅 maxFee
- **priority fee（tip）**：畀建塊者／驗證者嘅小費——決定你喺 mempool 排得幾前

`eth_feeHistory` 就係你嘅「市場深度 + 近期成交價」：睇近 N 塊嘅 baseFee 軌跡，同每塊實際成交 tip 嘅 **百分位**（p25／p50／p75…）。出金 oracle 用嚟揀 tip 檔同 maxFee 緩衝；卡單先再講 RBF bump（見 gas bump playbook）。

類比：撮合掛單——baseFee 似「最低保護價／指數」，tip 似「你願意加嘅滑點／優先費」；feeHistory 百分位似睇近幾檔成交嘅 tip 分佈，唔好只用「上一次成功出金嘅數」當永恆真理。

## 面試短答

- 呼叫大致：`eth_feeHistory(blockCount, newestBlock, rewardPercentiles)` → 每塊 `baseFeePerGas`、可選 `reward[]`（該塊 tip 百分位）、`gasUsedRatio`。
- **Tip**：按 urgency 對齊目標百分位（普通≈p50、加急≈p75／更高），再夾地板／天花板；空閑鏈都留最低 tip，避免「tip=0 理論可上、擁塞就永遠 pending」。
- **maxFeePerGas**：≈ `estimatedNextBaseFee × buffer + tip`。估下一塊可用最後一塊 baseFee ×（視 `gasUsedRatio` 升降）或短 EMA；buffer 視波動（常見 1.25–2× 量級，按鏈調）。
- **Tip 太低**：長時間無 receipt、同 nonce 排隊被後來高 tip 擠後；**maxFee 太低**：`baseFee + tip > maxFee` 時該塊根本收唔到你。
- **Tip／maxFee 過高**：上鏈快但燒 hot wallet ETH、提幣成本中心膨脹；要有熔斷同「加急檔」產品定價對齊。
- CEX：fee oracle 按鏈分開；出金前讀最新（忌長快取）；同「fee rate／urgency 佇列」映射成 percentile 檔，而唔係寫死 gwei。

## 常見追問（連答案）

**Q：baseFee 同 tip 邊個决定「卡唔卡」？**  
A：兩樣都可令你卡，機制唔同。`maxFee < 當前 baseFee + tip` → 交易喺該塊**無效**（唔係「慢」，係根本入唔到）。tip 夠但低於競爭對手 → 多數情況仍**可被收**，只係確認慢／擁塞時長期 pending。值班要分清：升級 maxFee 緩衝 vs 拉高 tip 搶優先。

**Q：`reward` 百分位點解唔用平均值？**  
A：單塊 tip 分佈極偏（有人塞超高 tip 搶頭位）。百分位穩過平均：p50 代表「一半交易 tip ≤ 呢個數」，更似「普通檔市價」；加急用更高分位。CEX 可維護 `urgency → percentile` 表，再加絕對地板（抗極端空閑）同天花板（抗被一筆怪 tx 拉爆）。

**Q：點判斷 tip 太低 vs 已經 overpay？**  
A：**太低**：超過 SLA（N 塊／M 分鐘）仍無 receipt，同時 feeHistory 顯示當前目標分位已高過你廣播時嘅 tip，或 mempool 觀察同類 tx 更高 tip 已上鏈。**Overpay**：確認極快但實際有效價長期遠高過同 urgency 嘅 p75／你設嘅成本預算——應下调檔位或收緊 buffer，唔好永遠「怕卡就 ×3」。監控：確認延遲分位、每筆有效 gas 價、bump 次數。

**Q：同 CEX「fee rate／urgency 佇列」點橋接？**  
A：產品層：普通／加急（甚至 VIP）→ 內部 `urgency` enum → oracle 映射 `targetPercentile`、`baseFeeBuffer`、`tipFloor/Cap`。帳務層：用戶付嘅提幣費要覆蓋預期 gas（按鏈），加急溢價對齊更高 tip 檔。技術層：同一套 `eth_feeHistory` 輸出，唔好每個佇列各自「拍腦袋 gwei」。卡單時 bump 仍用**同 nonce RBF**，urgency 只影响目標 tip／maxFee，唔好開新 nonce。

**Q：L2／Blob 時代仲要唔要睇 tip？**  
A：要，但參數同 L1 差好遠：好多 L2 tip 極低、確認主要由 sequencer 政策决定；L2 成本仲有 blob／DA 另一本帳（見 4844）。面試一句：feeHistory 思維通用（歷史 base／reward → 定價），但 **每條鏈獨立 oracle**，唔好把主網 gwei 抄去 Arbitrum／Optimism。

## 小練習

**題：** 熱錢包出金佇列：普通檔用 feeHistory 近 20 塊 p50 tip≈1.2 gwei，加急檔用 p75≈4 gwei。某筆普通出金已廣播 tip=1.2、maxFee=40；8 分鐘後仍 pending。此刻最新 baseFee≈36 gwei，p50 tip≈3、p75≈8。你會點診斷？下一步只准改 fee、不准開新 nonce——會點設？

**參考答案：**  
先拆兩截：`baseFee≈36` 已迫近 `maxFee=40`，若再升就變成「maxFee 不夠」而不只係 tip 慢；同時市價 tip 已由 1.2 升到 p50≈3，原 tip 明顯落後。同 nonce RBF：tip 對齊而家普通檔至少 p50（或因已超 SLA 臨時升一檔到 p75≈8），`maxFee ≈ estimatedNextBaseFee × buffer + tip`（例如估 base 再 ×1.25–1.5 + tip，確保明顯高過 40 同高過「base+tip」）。廣播新 hash，舊 hash=`replaced`；若連續 bump 觸成本熔斷 → cancel 清隊，而唔係 nonce+1 再出一次。
