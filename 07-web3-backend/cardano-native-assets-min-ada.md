# Cardano：Native Token + min-ADA（多資產 UTXO）充值 / 出金

## 人話
Cardano 同 BTC 一樣係 **UTXO**（一張張「找錢單」），但每張 UTXO 唔止裝 ADA，仲可以順便裝 **native token**（唔使合約，鏈本身識）。
有條硬規矩：**每個 output 一定要帶最少一啲 ADA（min-ADA）**，就算你只係想轉 USDM / SNEK 都一樣。好似寄包裹，就算入面係一張咭，都要貼最低郵費。

對 CEX 最痛兩點：
1. **用戶充 token，會順手帶埋 1 粒幾 ADA 入嚟** → 呢啲 ADA 入唔入帳？
2. **你幫用戶出 token，要自己貼 min-ADA** → 呢筆成本邊個畀？

## 面試短答
> Cardano 係 eUTXO 多資產模型，每個 output 包含 ADA 同可選 native token，token 以 `policyId + assetName` 唯一識別。協議要求每個 output 帶最少 ADA（按 output 大小 × `coinsPerUTxOByte` 計，帶 token 通常約 1–1.5 ADA）。
> 充值掃鏈用 db-sync / Ogmios+Kupo，按 `txHash#index` 冪等入帳，token 按 policyId 白名單匹配，唔信名；隨 token 帶入嚟嘅 ADA 要有明確政策（通常一齊入 ADA 帳）。出 token 時交易所要自己加 min-ADA，計入手續費報價。出金交易設 TTL（`invalid_hereafter`），過咗 slot 就永遠唔會上鏈，可以安全重建；確認數按風控取（例如 15–30+ 塊），大額再等多啲。

## 重點細節
- **Token 身份 = `policyId`（28 bytes hash）+ `assetName`（hex bytes）**。名字/ticker 任何人都可以抄 → 白名單只認 policyId+assetName，同 EVM 只認合約地址一樣道理。
- **min-ADA**：唔係固定 1 ADA，係 output 序列化大小 × 協議參數 `coinsPerUTxOByte`；output 入面 token 種類越多、assetName 越長、有 datum → 要更多 ADA。用 SDK（如 cardano-serialization-lib / CSL、Lucid）計，唔好寫死。
- **手續費**：線性 `minFeeA × tx大小 + minFeeB`，交易越大越貴；UTXO 太碎 → input 多 → 貴 + 可能超最大交易大小。
- **地址**：Shelley `addr1...`（bech32），base address = payment credential + stake credential；enterprise address 冇 stake 部分。HD 路徑 CIP-1852：`m/1852'/1815'/account'/role/index`。出金校驗：bech32 checksum、network（`addr` vs `addr_test`）、header 類型。
- **Stake 部分**：充值地址如果共用同一 stake key，所有地址嘅 ADA 會一齊計入 delegation（可以收 staking 獎勵）；獎勵喺 reward account，唔係 UTXO，提取要專門 withdrawal。
- **Finality**：Ouroboros 係概率性，可能淺 reorg；理論上 `k=2160` 塊後先絕對不可逆，實務交易所按風險設確認數。
- **TTL（validity interval）**：出金一定要設 `invalid_hereafter`；廣播後超時唔知結果 → 等過咗 TTL slot 再查，未上鏈就一定作廢，先可以用同一批 input 重新建交易。
- **腳本地址 / datum**：從 DeFi 合約地址出嚟嘅充值都係普通 UTXO 轉入，照計；但用戶要求「出金去腳本地址」通常要 datum，否則錢可能鎖死 → 出金拒絕或警告腳本地址（payment credential 係 script hash）。

## 常見追問（連答案）
**Q1：用戶充 100 USDM，output 夾埋 1.3 ADA，點入帳？**
A：兩個資產分開入帳：USDM 100 入 USDM 帳，1.3 ADA 入 ADA 帳（冪等鍵 `txHash#outputIndex + asset`）。要喺產品寫明，同埋 ADA 充值最小額要覆蓋呢個情況（唔好因為低過 ADA 最低充值就吞咗用戶錢，可以累計或者照入）。歸集時 token 同 ADA 一齊搬。

**Q2：出 token 嘅 min-ADA 邊個畀？**
A：兩種做法：(a) 交易所貼，計入 token 提幣手續費報價（最常見，用戶體驗好）；(b) 明確「出 token 會連帶 X ADA 一齊轉畀你」，從用戶 ADA 餘額扣。重點係報價時用 SDK 按真實 output 計 min-ADA，唔好寫死 1 ADA，否則交易 build 唔過。

**Q3：點解唔可以超時就即刻重發？**
A：UTXO 模型下重發同一批 input 嘅另一筆交易，最多一筆成功（double spend 被拒），本身唔會出兩次錢；但如果重建時換咗 input，就可能兩筆都上鏈 → 出兩次。所以：要麼重送同一份已簽好嘅 tx（同 txHash），要麼等 TTL 過咗確認冇上鏈先重建。同 BTC/Sui「重送同一份簽好 bytes」一個道理。

**Q4：Token 掃鏈要注意咩？**
A：一個 output 可能有多種 token（包括垃圾空投 token）；只處理白名單 policyId+assetName，其他忽略但保留紀錄。decimals 唔喺鏈上強制，要從 token registry / metadata 取並人工確認後存配置。

## 小練習（題目 + 參考答案）
**題：** 你哋熱錢包要批量出 50 筆 SNEK，一筆交易 50 個 output。上線後發現交易 build 失敗 / fee 好貴，可能原因同改法？

**參考答案：**
1. 每個 output 都要 min-ADA（50 × ~1.2 ADA ≈ 60 ADA），熱錢包 ADA 唔夠 → 先檢查 ADA 餘額，min-ADA 用 SDK 逐個計。
2. 熱錢包 UTXO 好碎，input 太多 → 交易超 max tx size、fee 線性上升 → 定期 consolidate UTXO，batch 細啲（例如每筆 10–20 個 output）。
3. 任何一個 output 地址錯（network 唔啱 / 腳本地址）會令成筆失敗 → 出金前逐個地址校驗，壞嘅剔出單獨處理，唔好拖死成批。
4. 每筆設 TTL，記錄 txHash，超時按 TTL 規則處理。
