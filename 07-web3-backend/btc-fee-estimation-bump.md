# BTC 手續費估算、CEX 代付與 fee bump（RBF / CPFP）

> 銜接：[BTC UTXO 充值／出金](./btc-utxo-deposit-withdrawal.md) 講 UTXO 同 RBF/CPFP 概念；[提現 Gas Bump 實務](./withdrawal-gas-bump-ops.md) 講 EVM 側。今日講 **vsize × 费率、平台代付、估算過低點救**。

## 人話

BTC 礦工費唔係「固定 gas × price」，而係：

**fee（sats）≈ vsize（vB）× fee rate（sat/vB）**

- **vsize** = weight / 4（BIP 141）。Legacy input/output 大；**SegWit**（bc1q）witness 計 weight 有折扣，同樣金額 SegWit 通常平啲。
- 一張 tx 有幾多 **input**（要簽幾粒 UTXO）、幾多 **output**（用戶 + **找零 change**）直接決定 vsize；找零多一個 output 就貴一截。

**CEX 代付（平台付 miner fee）**：出金由熱錢包 UTXO 組 tx，fee 從平台池扣。產品二選一（或混合）：

- 用戶填「到帳數量」，fee 另外由平台 absorb 或從另一條規則扣；
- 用戶填「扣帳數量」，到帳 = 扣帳 − 估 fee（要估準，否則爭議）。

用戶唔需要自己準備 sat 做 fee，但後端要 **估 rate、揀 UTXO、處理卡住**。

**估算來源**（可以組合用）：

- **`estimatesmartfee`**（bitcoind）：按目標確認 block 數（例如 1 block vs 6 blocks）估 sat/vB。
- **mempool.space / 第三方 API**：睇當前 mempool 分位、下一 block 預測。
- **自建 mempool 視圖**：同自家節點 policy 一致，適合大批量出金。

目標 block 愈少 = rate 愈高 = 愈快上鏈。尖峰要留 **buffer**（例如估 20 sat/vB，實際廣播 25–30），避免「差一啲」卡幾個鐘。

**Underfee 卡住**：

- **RBF（Replace-By-Fee）**：原 tx 建時 input `sequence < 0xfffffffe`（opt-in RBF）。用 **同一批 input**、更高 fee 重簽新 tx → 舊 tx 被替換。CEX 出金單要記 **replacement txid**，舊 id 標 `replaced`。唔可以用新 UTXO 再出一次（雙花風險）。
- **CPFP（Child-Pays-For-Parent）**：父 tx 冇開 RBF 或唔想動 input 時，用父 tx 某個 **output**（通常係找零返熱錢包）做 input，開 **高 fee 子 tx**，礦工為收子 fee 會打包父+子。要確保子 tx fee 夠「拉動」整個 package 的 vsize。

**批量出金**：一 tx 多個 output → fee 分攤，平好多；但一旦 underfee，**bump 複雜度上升**（成批人等同一張 tx、RBF 要成張替換、CPFP 要夠找零 output）。

**Dust / 細找零**：太細嘅 change（P2WPKH 約 294 sats 一類 dust 規則）可能唔經濟或節點唔轉發。Coin selection 要避免「為咗湊數造 uneconomic change」；**低峰期 consolidate UTXO**，減少將來 input 數同 vsize。

## 面試短答

- Fee = **vsize × sat/vB**；SegWit 細、input 數同 change 影響大。
- **代付** = 平台熱錢包付 fee；產品定 net/gross 顯示。
- 估算：`estimatesmartfee` / mempool 分位 / 自建 mempool；按目標確認 block 揀 rate + buffer。
- 卡住：**RBF**（同 input、更高 fee）優先；唔得再 **CPFP**（子 tx 付父）。
- 批量省 fee 但 bump 難；dust 同 consolidate 係長線成本。

## 常見追問（連答案）

1. **SegWit 同 legacy 對 fee 估算有咩影響？**
   答：估算仍係 sat/vB，但 **建 tx 時 vsize 唔同**。10 個 legacy input 可能幾百 vB；SegWit input 細好多。估 fee 前要先 **simulate 交易大小**（coin selection 定咗 input/output 數），唔好只背一個 global sat/vB 乘固定常數。

2. **代付時用戶見到咩？**
   答：視產品。常見「提 1 BTC，鏈上到 1 BTC，fee 平台包」或「提 1 BTC，扣 1 BTC + fee」。後者要廣播前 **fee 上限** 同 **估 fee 滑點** 寫清楚；估太低要麼平台補差（RBF bump 成本），要麼延遲出金唔夠 fee 拒絕。

3. **RBF 同 CPFP 點揀？**
   答：出金 tx 預設 **opt-in RBF** → 卡住用 RBF 最乾淨（同一 business id、同一批 UTXO）。父 tx 冇 RBF、或 input 唔敢動（已簽死）→ 用返熱錢包找零 output 做 CPFP；確認找零 value 夠大，子 tx fee rate 要 cover 父+子總 vsize。

4. **批量出金 50 個地址一 tx，fee 估低卡咗點？**
   答：成批 pending；**一次 RBF** 替換成張 tx（fee 可能好貴，因 vsize 大）。運維上要 **batch size 上限**、尖峰 **預先提高 rate**、或拆成多 tx 分散風險。唔可以為「救一批」再開 50 張新 tx 用同一批 UTXO。

5. **estimatesmartfee 返 1 block 同 6 block 差好遠，點揀？**
   答：睇 SLA：提現「10 分鐘內」用偏 1–2 block 分位 + buffer；非急用 6 block 慳成本。尖峰可以 **分 tier**（VIP 用 fast fee）。同時睇 mempool 是否空虛（估太高等於送 miners 小費）。

## 小練習（附參考答案）

**題：** 熱錢包要出 0.5 BTC 俾用戶，coin selection 揀 3 個 SegWit input + 1 個 output 俾用戶 + 1 個 change，simulate 得 **vsize = 250 vB**。mempool 建議下一 block **30 sat/vB**，你哋政策 +20% buffer。應鎖定幾多 fee？若只付 5000 sats 會點？

**參考答案：** rate ≈ 30 × 1.2 = 36 sat/vB；fee ≈ 250 × 36 = **9000 sats**（可略向上取整）。若只付 5000 sats ≈ 20 sat/vB，likely underfee，tx 長期 mempool 排後面；用戶投訴後用 **RBF** 同 3 個 input 重簽，fee 提到當前目標 rate。5000 sats 唔夠時唔好再開新 tx 用其他 UTXO 出 0.5 BTC。

**題：** 出金 tx 冇開 RBF，卡咗；tx 有 output：vout0 用戶 1 BTC，vout1 找零返熱錢包 2 BTC。點 CPFP？

**參考答案：** 用 vout1（2 BTC 找零）做 input 開子 tx（例如再轉去熱錢包另一地址或 consolidate），子 tx 設 **高 sat/vB**，令 package（父 250 vB + 子 ~150 vB）對礦工夠吸引。出金單關聯父 txid；子 tx confirmed 後父一齊確認。確保 vout1 未被其他 spend 鎖定，且子 fee 足夠拉動父 tx。
