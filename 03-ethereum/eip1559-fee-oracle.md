# EIP-1559 Fee Oracle（CEX 出金 Gas 定價）

## 人話

舊式交易只塞一個 `gasPrice`。EIP-1559 拆成：

- **base fee**：協議燒咗，跟區塊擁塞上下浮動  
- **maxPriorityFeePerGas（tip）**：畀驗證者／建塊者加速打包  
- **maxFeePerGas**：你肯付嘅上限 ≈ tip + 你預估會碰到嘅最高 base fee

CEX 出金服務要有 **fee oracle**：讀 `eth_feeHistory`／最新 base fee，估下一兩個區塊，再加安全邊際，寫入 `maxFee`／`maxPriorityFee`，否則提幣會卡 mempool 或者俾貴。

## 面試短答

EIP-1559 下，有效 gas 價 ≈ `min(maxFeePerGas, baseFee + priorityFee)`。  
出金服務唔好寫死 gasPrice：應用 fee history 預測 base fee，設合理 tip 同 maxFee 緩衝，再配合 stuck-tx 時用 **同 nonce RBF** 提高 tip／maxFee。  
CEX 還要把 gas 成本計入提幣費／內部成本中心，並按鏈分開 oracle（ETH、L2 參數唔同）。

## 常見追問（連答案）

**Q: maxFeePerGas 設太低會點？**  
A: 若 `baseFee + tip > maxFee`，交易喺該區塊無效／唔會被收，可能一直 pending。用戶以為「已廣播」但其實永遠上唔到鏈，要 RBF 或等 base fee 回落。

**Q: tip 設 0 得唔得？**  
A: 理論上部分空閑區塊可能仍被收，但擁塞時優先級極低，CEX 出金通常會設一個最低 tip（或按 urgency 分檔：普通／加急），保證可預期打包時間。

**Q: 點樣做一個簡單 fee oracle？**  
A: 拉 `eth_feeHistory`（例如最近 5–20 blocks 的 baseFee + reward tip）、用 short EMA／percentile 估下一塊 baseFee，再乘緩衝（如 1.2–2× 視波動）、加目標 tip；輸出 `maxPriorityFeePerGas` 同 `maxFeePerGas = estimatedBaseFee * buffer + tip`。出金前再讀一次 latest，避免用過期快取。

**Q: 同 legacy gasPrice 點共存？**  
A: 有啲鏈／錢包仲用 legacy。後端應按鏈能力分支：1559 填 `type=2` 字段；legacy 只填 `gasPrice`。簽名同 RBF 規則要一致，唔好混用兩套字段導致節點拒收。

## 小練習

**題：** 出金交易 pending 10 分鐘，監控顯示 base fee 已升過你當時嘅 maxFee。下一步點做？（一句流程）

**參考答案：**  
用**同一個 nonce** 重簽一筆更高 `maxFeePerGas`／`maxPriorityFeePerGas` 的替代交易並廣播（RBF）；**禁止**開新 nonce 再出同一筆，否則可能雙花／亂序。同時更新內部狀態為 `replaced`，繼續追蹤新 tx hash。
