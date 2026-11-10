# ERC-4626 金庫份額：CEX 上架／理財對帳

> 銜接：stETH／wstETH 係「協議自訂嘅 shares 模型」，見 [lst-rebasing-steth-wsteth.md](./lst-rebasing-steth-wsteth.md)。ERC-4626 把同一類概念做成標準接口。sDAI／sUSDS 等收益代幣、Yearn／Morpho 類金庫多數都跟呢套。wstETH 更早出現，**唔一定**實作 4626，但換算思路好似。

## 人話（從你熟的 CEX）

用戶買「會生息嘅美元幣」或者你把理財資金存進鏈上金庫時，鏈上通常唔係每天改佢持有嘅枚數（咁做會似 stETH，掃描同訂單簿都痛），而係發給佢一種 **share token**：枚數不變，每一枚兑返底層資產（USDC、DAI、ETH）嘅數量慢慢增加。

ERC-4626 就係呢種「代幣化金庫」嘅標準。你可以當成基金嘅份額：`deposit` 用底層資產申購份額，`redeem` 用份額贖回資產；淨值大概係 `totalAssets / totalSupply`。對 CEX 嚟講，最易踩嘅坑係把 share 當成 1:1 嘅底層幣入帳、用即時 `previewRedeem` 當預言機報價，或者對一個幾乎空嘅新金庫做大額申購，被人用「捐錢抬高匯率」嘅 inflation／donation attack 把份額稀釋到零。

## 面試短答

ERC-4626 金庫本身係 ERC-20（shares），另加一組申贖接口：

- `asset()`：底層資產合約
- `totalAssets()`：金庫認為自己值幾多底層（可以包括策略內部位，唔只係保險庫上嘅閒置餘額）
- `convertToShares`／`convertToAssets`：報價用換算，**容許不精確**，規範甚至鼓勵用時間加權等更穩陣嘅實作
- `previewDeposit`／`previewMint`／`previewWithdraw`／`previewRedeem`：盡量接近「而家真係執行會點」，因此**可被當下狀態操縱，唔適合單獨當公平價預言機**
- `deposit(assets, receiver)`／`mint(shares, receiver)`／`withdraw(assets, receiver, owner)`／`redeem(shares, receiver, owner)`
- `maxDeposit`／`maxMint`／`maxWithdraw`／`maxRedeem`：暫停、存款上限、流動性唔夠時可以係 0

規範要求四捨五入偏向金庫：用戶存入時拿到嘅 shares 向下取整，贖回時拿到嘅 assets 向下取整。事件有 `Deposit`、`Withdraw`，另外仲有普通 `Transfer`。

CEX 含義：

1. 資產 key 用 `(chainId, vault)`，同底層 `(chainId, asset)` **分開**。用戶充 100 個 share ≠ 入帳 100 USDC。
2. 展示可以同時顯示 share 餘額同 `convertToAssets` 估值，但撮合、出金、負債請以 share 枚數（或你內部認嘅 shares）為準；估值只係展示／風控。
3. 代客理財：用戶帳本記「認購嘅底層」定「持有嘅 share」要選一條，並用狀態機覆蓋 `deposit`／異步贖回；鏈上持有嘅係 share，對帳要用 share 餘額同 `totalAssets` 估值兩邊。
4. 上架前除咗普通 token checklist，仲要看：`totalAssets` 點計（會唔會被捐贈或預言機即時操縱）、有冇虛擬 shares／decimals offset 或死份額防 inflation attack、`maxDeposit`/`maxRedeem` 行為、係咪可升級、管理人可唔可以改策略／暫停。
5. 空金庫或極低 TVL 唔好接大額：經典攻擊係先存 1 wei 再直接轉一大筆底層進金庫抬高匯率，下一個存款者因向下取整只拿到 0 或極少 share。OpenZeppelin 用虛擬資產／虛擬 shares（`totalAssets + 1`、`totalSupply + 10^offset`）令攻擊好難賺錢，但唔係每個金庫都有。
6. 部分金庫係異步贖回（例如 ERC-7540 擴展，或者 RWA／有鎖定期嘅實作）：`requestRedeem` 之後先排隊，唔能假設 `redeem` 即時有流動性。狀態機類似跨鏈：已請求未到帳嘅資產唔能再墊出金。

## 常見追問（連答案）

**Q1：4626 share 同 stETH 有咩分別？**  
A：兩者都係「份額 × 匯率 = 資產」。stETH 把匯率變化表現成 **token 餘額自己變**（rebasing，冇 Transfer）；4626 同 wstETH 一樣，**token 餘額不變、匯率變**。所以 4626 掃描可以當普通 ERC-20 聽 `Transfer`；你唔會因為「派息」而喺對帳見到憑空多出來嘅枚數。變嘅係每枚值幾多底層。把 4626 當 rebasing 幣去每日加減用戶枚數，係做錯模型。

**Q2：點解唔可以用 `previewRedeem(1e18)` 當標價喂給撮合或抵押？**  
A：`preview*` 追求「依家執行大概得幾多」，金庫閒置資金、捐贈、閃電貸、同一塊內嘅存款都可以即時改 `totalAssets`。規範明確話 `preview` 可以對操縱敏感，`convert*` 先容許做成更穩陣嘅預言機。CEX 抵押／強平／財務估值應視情況用：獨立市場價、協議提供嘅保守匯率，或延遲／加權後嘅 `convertToAssets`，再加髮切。訂單簿如果買賣嘅係 share 本身，就用 share／share 或 share／USDT 嘅市場價，唔好把 NAV 當成一定成交得嗰個價。

**Q3：inflation／donation attack 具體點發生？上架點防？**  
A：空金庫 `totalSupply = 0` 時，攻擊者先 `deposit` 極少資產拿到幾乎全部 share，再**直接 `transfer` 一大筆底層給金庫合約**（唔經 `deposit`，所以唔增發 share）。匯率被抬到極高，下一個存款者 `convertToShares` 向下取整變成 0 或極少，資產卻留在金庫裡，贖回權喺攻擊者。防法：拒絕 TVL 過低或未初始化嘅金庫；檢查實作有冇虛擬 shares／decimals offset、有冇種子存款鎖在金庫；代客 `deposit` 時用有 slippage 下限嘅 router（實際拿到嘅 shares ≥ 預期）；小額試存之後對 `balanceOf` 同 `previewRedeem`。

**Q4：用戶在交易所「贖回 100 USDC」，鏈上應該叫 `withdraw` 定 `redeem`？**  
A：用戶要準確嘅係底層數量就用 `withdraw(assets, …)`（會燒對應 shares，shares 向上取整偏向金庫）；用戶要贖回全部份額就用 `redeem(shares, …)`。之前一定要讀 `maxWithdraw`／`maxRedeem`，回 0 就唔好送交易。`eth_call` 預演，核對實際 assets／shares 同報價；receipt 要以 `Withdraw` 事件同餘額差為準，唔好假設參數就係到帳數。

**Q5：代客把用戶 USDC 存進金庫，帳本記 100 USDC 定記 shares？**  
A：兩種產品模型，唔好混：  
- **現金增值型**：用戶看見 USDC 餘額慢慢加，內部你握 share，要每日／每小時按 `convertToAssets` 應計，贖回失敗、暫停、虧損都要有撥備同披露。  
- **份額型**：用戶看見同鏈上一樣嘅 share 枚數，盈虧體現在淨值，對帳最乾淨。  
無論邊種，鏈上對帳主鍵係 share 餘額；只對 USDC 總額會喺淨值上升時出現「鏈上（估值）多於帳本」或相反，原因好似 stETH rebase，但枚數層面唔會自己變。

**Q6：`totalAssets()` 係唔係就等於金庫合約嘅 `asset.balanceOf(vault)`？**  
A：唔一定。好多金庫會把資產劃去貸款、LP、策略合約，`totalAssets` 係會計數而唔係單一 `balanceOf`。呢個數可以滯後、可以依賴預言機、可以被高估。對帳要：托管地址持有嘅 share × 你信任嘅估值來源，再另做「策略披露 vs 可即時贖回流動性」監控。`maxRedeem` 遠低於 `convertToAssets` 總額，代表有部份並非即時流動。

## 小練習（題目 + 參考答案）

**題目：** 產品要上架一個新嘅「USDC 金庫份額」並開 Earn：用戶充 USDC，後端幫佢 `deposit` 進金庫，帳本直接記「USDC 100」同鏈上 USDC 1:1。報價服務每塊呼叫 `previewRedeem(shares)` 更新淨值。上線首日金庫 TVL 幾乎係 0。安全複盤時，有人質疑空金庫同報價來源。你會點改設計，先准上線？

**參考答案：**

- **資產分離**：用戶若持有嘅係 share，帳本應記 share（或獨立 earn 倉位），唔好寫成可用現貨 USDC。否則用戶以為可以即時當 100 USDC 現貨出金，但其實要 `redeem`，仲可能被暫停或排隊。
- **拒絕空金庫／低 TVL**：等有種子流動性，或確認合約用咗虛擬 shares／decimals offset，並且你用帶 slippage 保護嘅 deposit（實際 shares < 預期就 revert）。上架 checklist 加一項：fork 模擬「先捐一筆再 deposit」會唔會拿到 0 share。
- **報價**：撮合同抵押唔好直接吃每塊 `previewRedeem`。Share 現貨用市場價；Earn 展示 NAV 用 `convertToAssets` 或協議保守匯率，加更新頻率同髮切；異常跳升／跳跌熔斷申贖。
- **申贖路徑**：存款後記錄實際 mint 出嚟嘅 shares（`Deposit` 事件）；贖回先 `maxRedeem`，再 `previewRedeem`，上鏈後以事件同餘額差入帳。異步金庫要加 `REQUESTED → CLAIMABLE → CLAIMED`，在途唔可再墊其他出金。
- **對帳**：鏈上 share 餘額 vs 用戶 share（或內部倉位）合計；另外用 NAV 估底層敞口，但唔把估值差當成「無主 USDC 充值」。
- **回歸**：空金庫 donation 場景下代客 deposit 必須失敗或拿到預期 shares；NAV 被單塊操縱時抵押／報價唔跟住爆倉或錯價成交。
