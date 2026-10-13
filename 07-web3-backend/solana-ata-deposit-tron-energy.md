# Solana ATA 充值 + TRON Energy / Bandwidth

> 銜接：[Solana/TRON vs EVM 帳戶模型](./solana-tron-vs-evm-account-model.md)、[歸集 gas 策略](./sweep-gas-strategy.md)。

## Part A：Solana ATA（Associated Token Account）

### 人話
Solana 上「用戶充值地址」通常係錢包地址（owner），但 USDC 其實係打入**另一個地址**：由 `(wallet, token program, mint)` 推導出嚟嘅 ATA（一種 PDA，固定可算）。
CEX 類比：你畀客一個「客戶號」，但錢實際入「客戶號 × 幣種」嗰個子帳戶。子帳戶未開，錢就入唔到。
- ATA 要先建立，建立要付 **rent-exempt 押金**（token account 約 0.002 SOL），通常由**發送方錢包自動建**（create-idempotent 指令）。
- token account 入面記住 `mint`、`owner`、`amount`；**判斷係咪我哋嘅錢，睇 owner 同 mint，唔係睇地址長咩樣。**

### 面試短答
我會畀用戶錢包地址做充值地址，後端為每個 (deposit wallet, 支持嘅 mint) 預先算好 ATA 地址。掃描用兩條路：盯 ATA 簽名（`getSignaturesForAddress`）+ 掃塊用 `pre/postTokenBalances` 按 owner+mint 對帳兜底。入帳金額用**餘額差額**，唔用指令入面寫嘅 amount（Token-2022 有 transfer fee）。`finalized` 先 credited。歸集時熱錢包做 fee payer，deposit 地址唔使預先打 SOL。

### 常見追問（連答案）

**Q1：用戶直接打去錢包地址，ATA 未建，會點？**
A：大部分錢包會自動幫收款方建 ATA（發送方付押金）。如果發送方工具唔幫建，交易會失敗、錢唔會走（唔會「消失」）。CEX 可以預先幫熱門幣建好 ATA，代價係每個地址每隻幣 ~0.002 SOL 押金；常見折衷：只為主流幣（USDC/USDT）預建，其他靠發送方建。

**Q2：用戶打咗去一個「唔係 ATA」但 owner 係我哋嘅 token account？**
A：Solana 允許任何人開一個 owner 係你嘅普通 token account。如果你只盯 ATA，就會**漏單**。所以要有兜底：定期 `getTokenAccountsByOwner(我哋地址)` 或掃塊時用 `postTokenBalances.owner` 對帳，發現非 ATA 嘅都入帳 + 歸集。

**Q3：假 USDC 點防？**
A：任何人都可以 mint 一隻叫 USDC 嘅幣。只認 **mint 地址白名單**（同 EVM 認合約地址一樣），symbol/name 一律唔信。

**Q4：Token-2022 有咩要注意？**
A：同一錢包 + 同一 mint，用 Token-2022 program 推導出嘅 ATA **唔同**於舊 Token program。另外有 transfer fee、凍結等 extension：入帳用到帳差額；上架前要查 mint 有冇 freeze authority / transfer hook（類似 EVM 黑名單 / 稅幣）。

**Q5：歸集後要唔要 close ATA 拎返押金？**
A：可以 close 拎返 ~0.002 SOL，但下次用戶再充，要靠發送方重新建；大部分 CEX 會保留主流幣 ATA，用戶體驗優先。冷門幣 / 長期唔活躍地址先 close。

## Part B：TRON Energy / Bandwidth

### 人話
TRON 手續費唔係「每筆付 gas」，而係兩種資源：
- **Bandwidth**：按交易 bytes 計，每個帳戶每日有少量免費額度；普通 TRX 轉帳通常靠呢個。
- **Energy**：執行智能合約（例如 USDT TRC-20 transfer）用。
冇資源就**燒 TRX** 抵數。資源嚟源：質押 TRX（Stake 2.0）、或者由另一個帳戶**代理（delegate）**畀你、或者喺市場租。
CEX 類比：燒 TRX = 按次付費；質押 = 買月費流量包；代理 = 公司總戶分流量畀子帳戶。

### 面試短答
TRON 出金同歸集，我會由熱錢包質押 TRX 攞 energy，日常出金用自己 energy；歸集時 delegate energy/bandwidth 去 deposit 地址，掃完 undelegate，避免逐個地址打 TRX（gas 站模式）。USDT 轉去「冇 USDT 餘額」嘅地址 energy 大約要雙倍（要新開 storage slot），出金成本估算要分開計。交易設 `fee_limit` 封頂，失敗都會收費，所以 receipt 要檢查。

### 常見追問（連答案）

**Q1：點解 USDT 轉去新地址貴好多？**
A：TRC-20 `balanceOf` 由 0 變非 0 要寫新 storage slot，energy 約 13 萬 vs 已有餘額約 6.5 萬。所以出金手續費估算唔可以用平均值，要按收款地址有冇餘額分開。

**Q2：歸集 TRON deposit 地址仲有咩坑？**
A：（1）地址未激活：只收過 TRC-20、從未收過 TRX 嘅地址可能未激活，要先激活（例如由熱錢包打少少 TRX）先發得交易；（2）就算 delegate 咗 energy，仲要有 bandwidth（免費額度或代理）；（3）delegate/undelegate 本身都係交易，有成本，所以要設歸集閾值，細額唔掃。

**Q3：`fee_limit` 設幾多？**
A：係「最多肯燒幾多 TRX」嘅上限。設太低 → `OUT_OF_ENERGY` 失敗但照收費；設太高 → 遇到惡意 / 異常合約燒大錢。按幣種歷史用量 × 安全系數設，並監控失敗率。

## 小練習（題目後即附參考答案）

**題 1（Solana）：** 客投訴：「我打咗 1,000 USDC 去你哋地址，Solscan 顯示成功，但帳戶冇入帳。」你點查？

**參考答案：**
1. 用 txid 睇 `postTokenBalances`：收款 token account 嘅 `owner` 係咪我哋 deposit 地址、`mint` 係咪白名單 USDC。
2. 如果 owner 啱、但嗰個 token account **唔係我哋預算嘅 ATA** → scanner 漏咗（只盯 ATA），用 owner 兜底掃描補入帳，並修 scanner。
3. 如果 mint 唔係正牌 → 假幣，唔入帳。
4. 如果 owner 根本唔係我哋 → 用戶打錯地址，走錯充找回 / 客服流程。
5. 確認 commitment：未到 `finalized` 就顯示「確認中」。補入帳用冪等鍵 `(chain, signature, 指令 index / account)` 防重。

**題 2（TRON）：** 你有 5 萬個 TRON deposit 地址，每日有 2,000 個收到 USDT，要歸集到熱錢包。設計 gas 方案。

**參考答案：** 熱錢包（或專用資源帳戶）質押足夠 TRX 攞 energy + bandwidth；超過閾值嘅地址先歸集：檢查激活（未激活先打少量 TRX 激活）→ `delegateResource` energy（同 bandwidth）畀該地址 → 該地址簽 USDT transfer 去熱錢包 → 確認 `SUCCESS` → undelegate 收返資源。資源唔夠時可短期租 energy 或燒 TRX，`fee_limit` 封頂；帳本只記內部「歸集成本」，唔改用戶餘額。
