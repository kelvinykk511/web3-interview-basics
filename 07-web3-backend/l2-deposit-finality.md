# L2 充值最終性（Arbitrum / Optimism / Base）

## 人話
L2 出塊好快，但「L2 sequencer 話有」≠「以太坊 L1 認咗」。層次：
1. sequencer 軟確認（秒級，sequencer 可以重排/掛）
2. batch 已提交上 L1（tx data 喺 L1 上）
3. L1 batch 所在區塊 finalized（~13 分鐘）→ 基本不可逆
注意：7 日挑戰期係**提款返 L1** 用，唔係充值入帳要等。

## 面試短答
L2 充值我會按金額分級：小額用 L2 `safe`（batch 已上 L1）入帳，大額等 L2 `finalized`（對應 L1 finalized）。節點用自己跑或多供應商交叉驗證，sequencer 停機時暫停入帳而唔係盲等。

## 常見追問
- **Q：點解唔用 L2 確認數？** A：L2 無 PoW 式 reorg 概念，風險來自 sequencer 未上 L1；用 `safe/finalized` block tag 比數 block 準。
- **Q：sequencer 掛咗點？** A：監控 L2 head 停滯 → 該鏈充值顯示「延遲」、提現暫停；恢復後由游標補掃，冪等鍵 (chainId, txHash, logIndex)。
- **Q：要唔要等 7 日？** A：充值唔使；只有經官方橋由 L2 提返 L1 先要。

## 小練習
題：用戶喺 Base 充 50 萬 USDC，sequencer 已確認 2 秒，你入唔入帳？
答：唔即刻入。狀態 seen→confirming，等 `finalized`（約 15–20 分鐘）先 credited；期間 UI 顯示確認中。大額亦可人工複核。
