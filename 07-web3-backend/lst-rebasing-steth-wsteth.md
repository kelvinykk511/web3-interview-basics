# LST 生息代幣：stETH rebase vs wstETH（CEX 入帳／帳本）

> 銜接：非標準 ERC-20 有提過 rebasing／1–2 wei 捨入，見 [non-standard-erc20-token-listing.md](./non-standard-erc20-token-listing.md)、[token-listing-onchain-checklist.md](./token-listing-onchain-checklist.md)；交易所自己跑 validator 見 [eth-staking-product-backend.md](./eth-staking-product-backend.md)。今日專注 **用戶充進來嘅 LST 點記帳**，而唔係你自己去 stake。

Ethereum mainnet 常見合約（資產 key 仍然係 `(chainId, contract)`，唔好用 symbol）：stETH `0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84`，wstETH `0x7f39C581F595B53c5cb19bD0b3f8dA6c935E2Ca0`。

## 人話（從你熟的 CEX）

普通 ERC-20（USDT）入帳好直：掃到 `Transfer`，地址多咗幾多就入幾多，之後餘額唔會無端端自己變。stETH 唔係咁。佢代表你喺 Lido 池入面嘅份額（shares）：獎勵（或者罰則）到嘅時候，協議更新「每一份值幾多 ETH」，**所有持有人嘅 `balanceOf` 一齊加減，唔會逐個地址 emit `Transfer`**。

你可以想像成：用戶喺交易所持有一隻「會派息」嘅現貨。息唔係另一筆轉帳打入嚟，而係帳面單位自己脹（正 rebase）或者縮（負 rebase，例如 slashing 被社會化）。如果你個 scanner 只聽 `Transfer`，每日對帳就會見到「鏈上多咗一啲、帳本冇記」；罰則嗰日就會變成「鏈上少咗、帳本仲寫住舊數」——後者係缺口，嚴重過前者。

wstETH 係把同一份 shares 包成**餘額唔會自己變**嘅 ERC-20：你持有嘅枚數固定，變嘅係「1 枚 wstETH 兑幾多 stETH／ETH」。對撮合、掃描、出金嚟講乾淨好多，所以好多系統更願意只上 wstETH，或者內部用 shares／wstETH 記，對外先顯示 stETH。

## 面試短答

stETH 用 shares 記產權：`balanceOf(user) ≈ sharesOf(user) × totalPooledEther / totalShares`（實際用 `getPooledEthByShares`／`getSharesByPooledEth`）。共識層 oracle report（正常大約一日一次）更新匯率，持有人 shares 不變、`balanceOf` 變，**冇對應 Transfer**。轉帳本身亦可能有 1–2 wei 捨入，歸集時唔好當異常。

CEX 含義：

1. **入帳**以充值地址 `balanceOf`／`sharesOf` 差額為準，`Transfer` 只係線索；事件金額同餘額差可以差 1–2 wei。
2. **帳本最小單位最好記 shares**（或內部 wrap 成 wstETH），展示層再用當前匯率换成 stETH；淨係把充值當下嘅 stETH 整數鎖死，之後 rebase 會變成平台隱藏盈虧，負 rebase 仲會令鏈上少於負債。
3. **現貨產品要講清楚**：用戶餘額係唔係跟住協議 rebase 走。跟就每日（或每次 oracle report）按股份分配custody 地址嘅差額；唔跟就等於平台食息，產品同對帳都要明示，唔可以當對帳 bug 隱形食咗。
4. **協議匯率 ≠ 二級市場價**。`getPooledEthByShares` 接近「1 stETH ≈ 1 份協議內 ETH」（罰則後會低過 1），Curve／訂單簿可以脫錨。估值、抵押、強平唔可以用 1:1，要分 NAV 同市場價。
5. **原生贖回**行 Lido `WithdrawalQueueERC721`：`requestWithdrawals`／`requestWithdrawalsWstETH`（單筆最少 100 wei、最多 1000 stETH，再大要拆單），換成不可取消嘅 unstETH NFT，FIFO，等 oracle finalize 之後先 `claim` 到 ETH。隊列中嘅資產唔係流動庫存，唔好攞去墊其他用戶出金。NFT `owner` 先有權 claim，owner 填錯等於把提款權轉走。
6. 只上 wstETH 更穩：`wrap`／`unwrap` 用同一套 shares 公式；wstETH 餘額只喺轉帳、mint、burn 時變。用戶充 stETH 你唔可以靜默改寫成 wstETH 資產（除非產品寫明並得到同意）。

## 常見追問（連答案）

**Q1：點解只掃 Transfer 會漏數？漏嘅係用戶充值定係收益？**  
A：用戶鏈上 `transfer` 仍然有 `Transfer` event，充值本身掃得到（加減 1–2 wei）。漏嘅係 **rebase**：獎勵令所有托管地址 `balanceOf` 一齊升，罰則一齊跌，兩邊都冇 Transfer。對帳要用固定區塊高度讀 `balanceOf`／`sharesOf`，同帳本 shares 對上。多出嚟嘅正差額係待分配收益或平台自有，視產品而定；負差額要當缺口升級，唔好當掃漏一筆充值去「補入帳」畀某個用戶。

**Q2：帳本應該記 stETH 數量定記 shares？**  
A：托管視圖記 shares 最穩，因為 rebase 唔改 shares。展示「stETH 餘額」用 `getPooledEthByShares`。如果訂單簿要買賣「一枚 stETH」，成交結算仍然係轉 stETH 枚數，但每個用戶嘅可用餘額要喺 rebase 後重算，否則會出現「用戶合計 ≠ 鏈上托管」。wstETH 現貨就無呢個痛點：枚數穩定，價格／淨值另算。

**Q3：1 stETH 係咪永遠等於 1 ETH？**  
A：唔係。協議內匯率反映池內 ETH（含未完全社會化嘅罰則）對 shares 嘅價值，通常接近 1 但可以低過 1。二級市場仲可以因為流動性、脫錨交易而離開協議匯率。CEX 唔可以喺抵押、理財換算、財務估值度寫死 1:1；交易對用市場價，贖回／包裝用協議公式，兩條數要分開存。

**Q4：用戶要「unstake 返 ETH」，同喺現貨賣 stETH 有咩分別？**  
A：賣現貨係訂單簿或 OTC，即時、有市場價滑點。原生贖回係鎖 stETH／wstETH 進隊列、等協議有 ETH（buffer 或 validator exit）再 claim，時間不可控，但兑嘅係協議匯率而唔係訂單簿。後端要有狀態機：`REQUESTED → FINALIZED → CLAIMED`，持久化 `requestId`，監控 NFT owner；已 request 就唔可以當仲握住流動 stETH。超時只可以告警同等，唔可以取消重送同一筆（協議唔畀取消）。

**Q5：歸集掃到充值地址少咗 1–2 wei，要唔要當丢失？**  
A：stETH 轉帳按 shares 換算再 round，常見 1–2 wei 誤差，帳本要有 dust 容忍，對帳規則預先寫明。差額大過 dust 先升級。更乾淨係用戶充值後內部 wrap 成 wstETH 再歸集，後續轉帳唔再疊捨入。

**Q6：負 rebase（slashing 社會化）嗰日應該點做？**  
A：所有 stETH 托管地址 `balanceOf` 下跌但 shares 大致不變。如果用戶帳本記嘅係「跟協議走嘅 stETH」，要按新匯率調低顯示／可用（產品條款要先寫清）；如果帳本鎖死舊枚數，鏈上會少於用戶負債，呢個係資不抵債訊號，要暫停該資產出金同對帳，而唔係從別的用戶挪幣填數。

## 小練習（題目 + 參考答案）

**題目：** 你哋上架咗 stETH 現貨。Scanner 只認 `Transfer` 入帳。用戶 A 充入 10 stETH，入帳 10。三日後每日對帳發現熱錢包＋充值地址合計 stETH 比用戶負債多 0.004 stETH，團隊當「掃漏咗一筆細額無主充值」準備掛去平台收入。又過一星期，Lido 社會化一筆罰則，同一批地址合計少咗 0.02 stETH，用戶帳本總額冇改，對帳第一次出現鏈上少於負債。問題喺邊？點改？

**參考答案：**

- **誤判正差額**：多出嘅 0.004 好大機會係 rebase 收益，唔係無主充值。無主充值會有 `Transfer`（或至少有對應 tx／trace）；rebase 冇。要用 `sharesOf` 對托管地址：shares 冇增但 `balanceOf` 增，就是匯率變，唔好虛構一筆充值。
- **帳本單位錯咗**：用戶負債鎖死「充值當時嘅 stETH 枚數」，收益變成平台隱性收入，罰則變成平台隱性缺口。應改為記 shares（或內部 wstETH），展示用當前 `getPooledEthByShares`；正負 rebase 都跟住調整用戶可視餘額，或者產品明確「現貨唔派息、收益歸平台」並用單獨 PnL 科目，而唔係 silently 食正差額、負差額先爆雷。
- **對帳**：固定 block height，比對「鏈上 shares／stETH」vs「用戶 shares 合計＋平台自有」。正負差額都要解釋；負差額 fail-closed，暫停 stETH 出金，直到負債改寫或平台撥備補上。
- **產品**：現貨若不想處理 rebase，改上 wstETH，或充值時（經用戶同意）wrap 再記帳。
- **回歸**：模擬一次正 rebase——對帳解釋為收益而唔產生充值單；模擬一次負 rebase——用戶可視餘額或撥備科目要跟上，唔出現無主入帳、亦唔挪用其他用戶。
