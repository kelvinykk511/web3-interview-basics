# ERC-7683／意圖橋：CEX 充值入帳 vs 金庫調撥

> 銜接：官方 USDC 走 [CCTP](./usdc-cctp-cross-chain-rebalancing.md)，發行方 OFT／USDT0 走 [LayerZero OFT](./layerzero-oft-vs-cctp-rebalancing.md)。橋充值掃描同最終性見 [cross-chain-bridge-deposit.md](./cross-chain-bridge-deposit.md)。對手歸因（`from` 係通道而唔係經濟來源）見 [kyt-attribution-hot-wallet-taint.md](./kyt-attribution-hot-wallet-taint.md)。今日對焦 **filler／solver 墊付嘅跨鏈意圖（ERC-7683，Across 係生產實作）**，同官方 burn/mint 有咩唔同。

## 人話（從你熟的 CEX）

官方 CCTP／OFT 好似發行方自己「一邊燒、一邊印」，中間信 attestation 或該幣嘅 DVN 棧，原則上 1:1。**意圖橋**則像請一位做市商幫手：用戶喺源鏈鎖住 input，聲明「我想喺目標鏈把 output 打去某個收款地址」；**filler／relayer 先用自己嘅庫存在目標鏈墊付**，之後先喺源鏈（或佢指定嘅還款鏈）取回本金加費。

呢個模型快、唔使等官方最終性，但 **唔保證 1:1、唔保證一定成交**，超時會喺源鏈退款。用戶把 Across／聚合器嘅收款人填成你哋充值地址時，你見到嘅 `from` 好多時係 SpokePool 或 relayer，而唔係用戶自己——同「只信上一跳」嘅 KYT 陷阱同一類。

## 面試短答

1. **ERC-7683** 係跨鏈意圖嘅開放介面（Origin settler `open`／`openFor`，Destination settler `fill`），唔係一條新橋。訂單描述「鎖邊種、收邊種、去邊條鏈、收款人、截止時間」。**Across** 係而家常見嘅 settlement 實作之一，底下仍然係 V3 deposit／fill；標準事件（例如 `Open`／`Filled`）同 Across 自身事件（`FundsDeposited`／`FilledV3Relay`）可能同時出現，對帳要識兩邊，但 **入帳只認目標鏈已確認嘅到帳**。
2. **生命週期**：`OPEN`（源鏈已鎖 input）→ `FILLED`（目標鏈已付 output）→ filler 後期結算／還款；若喺 `fillDeadline` 前無人 fill，走 **源鏈退款**。可選 `exclusiveRelayer`＋`exclusivityDeadline`：窗口內只有指定 relayer 可填，過後先開放競爭。獨家窗口未過就當失敗，係誤判。
3. **金額**：`outputAmount` 通常 **小於** `inputAmount`（費、價差）。用戶充值以目標鏈 **實際收到** 為準，唔好按源鏈鎖倉金額入帳。資產 key 仍然係 `(chainId, contract)`，小心 output 其實係 bridged 變體而唔係原生 USDC／USDT0。
4. **CEX 入帳規則**：只在目標鏈看到打入充值地址嘅原生轉帳／`Transfer`，並且達到該鏈確認／最終性政策之後先 credit。**禁止**用源鏈 `open`／`FundsDeposited` 預入帳——未成交會喺源鏈退回，目標鏈從來冇貨。冪等鍵係目標鏈 `(txHash, logIndex)` 或等價到帳事實；`orderId`／`(originChainId, depositId)` 只用來追溯同客服，唔用來放行資金。
5. **KYT**：`from` 係 filler／SpokePool／settler 時，要升去做 **跨鏈追溯**，還原源鏈 depositor／經濟來源，唔好因為「官方橋／知名 relayer 低風險」就自動放行並 sweep 入主熱錢包（政策同 11-09 歸因一致）。`message` 非空可能觸發目標合約 callback，充值地址應避免執行任意 calldata。
6. **金庫調撥**：USDC 優先 CCTP，發行方官方 OFT 走 OFT，都係 1:1 burn/mint。意圖橋係 **墊付庫存＋超時退款**，唔適合作主路徑；緊急調撥都要按 `outputAmount`、deadline、未成交退款做狀態機，源鏈鎖定期間唔好把目標鏈庫存當已可用，更唔好按 1:1 入帳。

## 常見追問（連答案）

**Q1：用戶話「我已經喺以太坊 lock 咗，點解帳戶未到帳？」**  
A：先問清楚係意圖橋定官方 CCTP／OFT。若係意圖：源鏈 lock 只係 `OPEN`，要等目標鏈 fill 同確認。未到 `fillDeadline` 可以等或查有冇 exclusive window；過期未 fill 就應該喺 **源鏈** 退款，CEX 側本來就唔應該入帳。客服用 `orderId`／depositId 查狀態，但唔好因此手動預入。

**Q2：點解金庫唔用 Across 做日常調撥？**  
A：速度係 filler 庫存換嚟嘅，有費用、有未能成交同 settlement 合約風險，金額亦唔係 1:1。官方 CCTP／OFT 嘅擔保模型清楚（Circle attestation 或該幣 DVN 棧）。意圖橋最多當緊急流動性，而且要有 haircut、超時同「未 fill 就源鏈退回」嘅帳本狀態，唔好同官方路徑共用同一套「burn 完就等額 mint」假設。

**Q3：目標鏈已經見到一筆入帳，但源鏈訂單後來點解又退款？**  
A：正常填成嘅單唔會再退同一筆 input。若你看到退款，多半係 **另一筆未成交訂單**，或你把無關嘅 origin event 對錯單。入帳只綁目標鏈到帳；起源事件只做追溯。對帳時 `OPEN` 未 `FILLED` 計用戶喺源鏈嘅在途，唔計你哋交易所負債。

**Q4：`fillDeadline` 同 `exclusivityDeadline` 有咩分別？**  
A：`exclusivityDeadline` 只係「呢段時間內指定 relayer 獨佔」。過咗仍然可以喺 `fillDeadline` 前由其他 filler 成交。只有過咗 `fillDeadline` 未 fill，先進入退款。監控同客服唔好把獨家期結束當成訂單失敗。

## 小練習（題目 + 參考答案）

**題目：** 產品想快啲把 200 萬 USDC 由 Ethereum 調去 Base：走 Across，源鏈 lock 成功後 30 秒內 Base 熱錢包就見到 relayer 轉入 1,997,600 USDC。Job 用源鏈 lock 金額 200 萬把 Base 庫存標成可用。同時有用戶用同一個 Across 把資金提到自己嘅充值地址，scanner 見 `from` 係 SpokePool、KYT 分數低，就自動入帳並 sweep 入主熱。有咩問題？

**參考答案：**

- **金庫**：呢筆最多按 **實際收到嘅 1,997,600** 入在途／庫存，唔好寫 200 萬（差價係橋費）。更穩陣嘅主路徑係 CCTP（原生 USDC、1:1）。若堅持用意圖橋應急，狀態要等目標鏈確認（唔係源鏈 lock），並處理「未 fill 就源鏈退款、目標從未到帳」。
- **用戶充值**：到帳金額以 Base 實收為準係對嘅，但 **唔可以**因為 SpokePool 低風險就跳過歸因。要追溯源鏈 depositor；未放行資金唔好 sweep 入主熱，走隔離地址。credit 綁目標鏈到帳事實，唔綁 origin `open`。
- **回歸**：模擬「origin open 後超時退款、dest 從未 fill」——任何 job 都唔可預入帳或預增 Base 庫存；模擬「dest 已 fill 但 from 係 relayer」——入帳金額＝output，KYT 要有 origin 標籤。
