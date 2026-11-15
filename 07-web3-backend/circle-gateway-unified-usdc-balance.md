# Circle Gateway 統一 USDC 餘額（CEX 金庫調撥／即時出金）

> 銜接：CCTP 調撥見 [usdc-cctp-cross-chain-rebalancing.md](./usdc-cctp-cross-chain-rebalancing.md)，意圖橋唔好當成官方路徑見 [erc7683-intent-bridge-cex.md](./erc7683-intent-bridge-cex.md)。今日講 Circle **Gateway**：先把 USDC 存入官方 Wallet 合約並等最終性，之後用統一餘額喺支援鏈即時 mint 出金，唔使每次調撥中途再等源鏈最終性。

## 人話（從你熟的 CEX）

多鏈出金最煩係每條鏈都要預留一攤 USDC。CCTP 已經係官方 burn／mint，但每一筆仍然要喺「調撥途中」等源鏈最終性（Ethereum 同若干 L2 往往十幾分鐘）。Gateway 把呢段等待 **提前做完**：金庫用官方方法把 USDC 存入各源鏈嘅 **Gateway Wallet**（非托管託管合約），等 Gateway 按該鏈所需確認數認定存款已最終確定之後，呢筆錢進入 **統一餘額**（off-chain ledger，按鏈／代幣／地址記）。之後要喺邊條支援鏈出金，簽一個 **Burn Intent** 向 Gateway API 換 attestation（官方寫即時路徑 <500ms），再喺目標鏈 **Gateway Minter** 鑄出 USDC。你可以花「超過任何單一鏈上已存金額」嘅總額，唔使預先決定目的地。

同 CCTP 一樣係 1:1 對應 burn／mint，並唔係第三方池子墊付。分別係：最終性喺 **存款階段** 先付，轉出先至快。官方亦支援 EURC、cirBTC，但 **唔會幫你兌換資產**；每種資產一條統一餘額。資產同支援鏈以 [Gateway supported blockchains](https://developers.circle.com/gateway/references/supported-blockchains) 為準（文件寫緊 domain 編號同 CCTP 相同，**唔係 chainId**）。

## 面試短答

1. **必須走 Wallet 嘅存款方法。** 官方方法包括 `deposit`、`depositFor`、`depositWithPermit`（EIP-2612）、`depositWithAuthorization`（ERC-3009）。**用普通 ERC-20 `transfer` 打去 Gateway Wallet 地址會導致該筆 USDC 丟失**，唔會進入統一餘額。千萬唔好把 Wallet 合約當成用戶充值地址或「隨便轉入就入帳」嘅收款地址。
2. **未最終確定嘅存款唔係可用庫存。** Gateway 只會喺存款 **finalized** 之後先加統一餘額。文件嘅 Required block confirmations 寫緊：Ethereum 同若干以 ETH 塊為準嘅 L2（例如 Arbitrum、Base、OP、Unichain、World Chain）大約 **65 個 ETH blocks、約 13–19 分鐘**；Avalanche／Polygon／Solana 等快鏈可以係數秒。有 pending deposits API，**pending ≠ 可即時轉出**。另有 Fast Deposit（經 CCTP Fast，結算到 Avalanche 或 Polygon PoS）係另一條加速路徑，費用同適用範圍以官方為準。
3. **即時轉出要過兩關，入帳只認目標鏈 mint。** 用戶／金庫簽 Burn Intent（可加 delegate；EVM 亦支援 ERC-1271）→ `POST /v1/transfer` 拿 attestation＋operator 簽名 → 目標鏈 Minter mint 到 `destinationRecipient`。Attestation **約 10 分鐘過期**；過期未用，統一餘額會加返。庫存狀態要分「鏈上熱錢包／Gateway 在途存款／統一可用／mint 在途／目標鏈已到帳」。費用（跨鏈 transfer fee、源鏈 burn 嘅 gas fee，用 Forwarding Service 仲有 forwarding fee）從統一餘額扣，金額以 `/v1/estimate` 為準；**同源同目的鏈嘅提取唔收 transfer fee，只收 burn gas**。官方寫 Arc 作為源或目的時唔收 transfer fee。
4. **API 掛咗唔等於錢冇咗。** 非托管：冇用戶（或已授權 delegate）簽名，Circle 唔能自己燒走 Wallet 裡嘅幣。可以 `initiateWithdrawal`，等 **7 日** 再 `withdraw` 完成信任最小化提取。一經 initiate，統一餘額即減，呢 7 日唔能再當即時流動性。同鏈提取其實都係 burn＋mint，並唔係直接從 Wallet `transfer` 出嚟。
5. **同用戶充值通道分開。** Gateway 主要係金庫／出金流動性工具。用戶把 USDC 轉到 Gateway Wallet ≠ 充值成功（仲可能丟幣）。若產品要收「經 Gateway mint 入我哋地址」嘅款，只認 Minter 鑄到你指定收款地址嘅實收，並用 transfer spec／hook 追溯，唔好用源鏈存款預入。

## 常見追問（連答案）

**Q1：已經有 CCTP，仲要唔要 Gateway？**  
A：兩者都係 Circle 官方 1:1 路徑，用途不同。CCTP 適合單筆調撥、唔預先把庫存放進 Wallet。Gateway 適合事先完成最終性，之後喺支援鏈即時滿足出金，減少每條鏈都預留一大攤資金。唔好用 filler 橋去替代呢兩條官方路。支援鏈、domain、費用都以官方頁同 `/v1/estimate` 為準，唔好寫死。

**Q2：熱錢包 `balanceOf` 已經少咗，可唔可以當統一餘額已可用？**  
A：唔可以。`deposit` 成功只代表幣入咗 Wallet 合約；要等該鏈所需確認數、Gateway 觀察到 finalized 嘅 `Deposit`，`/v1/balances` 先係可花費額。之前只能記「Gateway 在途」。同樣，拿到 attestation 唔等於目標鏈已有幣，要等 mint 成功（或 attestation 過期回沖）。

**Q3：加咗 delegate 之後點管理風險？**  
A：Delegate 可以為該 Wallet 上嘅存款簽署轉出，相當於對呢筆統一餘額嘅高權限。要逐條已存款嘅鏈 `addDelegate`／`removeDelegate`。撤銷喺鏈上 finalized 之後 API 會停止接受新請求，但 **撤銷前已簽、尚未過期嘅 Burn Intent 仍然可以用嚟執行 burn**。金鑰輪換要當高權限操作，並監控 delegate 變更。

## 小練習（題目 + 參考答案）

**題目：** 金庫要把 500 萬 USDC 做成「各支援鏈即時出金」。Job 用普通 `transfer` 把 Ethereum 熱錢包 USDC 打進官方 Gateway Wallet 地址，並即時把 500 萬從「ETH 熱錢包」撥去「Base 可用庫存」。同時產品把同一個 Wallet 地址顯示俾用戶做「多鏈 USDC 充值」。有咩問題？點改？

**參考答案：**

- **普通 `transfer` 入 Wallet 會丟幣**，唔會進入統一餘額。必須用 `deposit`／permit／authorization 等官方方法；`depositFor` 要對準正確 depositor，唔好入錯科目。
- **唔可以存款當刻就撥 Base 可用**：先入「Gateway 在途」，等 Ethereum 約 65 blocks 最終性同官方餘額 API 確認，先入「統一可用」；真正出金要等目標鏈 mint 到指定地址先入該鏈庫存，並扣除 estimate 費用。Attestation 過期要回沖，唔好當成已到帳。
- **Wallet 合約唔能當用戶充值地址**：用戶直轉可能丟失，且即使正確 `deposit`，貨權係用戶自己嘅統一餘額，唔係交易所托管。用戶充值仍用你哋已監控嘅充值地址；若支援 Gateway 收款，只認 mint 到你地址嘅實收。
- **要有 API 故障預案**：`initiateWithdrawal` 後有 7 日延遲，期間呢筆唔能再用於即時出金。Delegate 權限同過期 intent 要納入熱錢包風控。
