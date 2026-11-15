# ERC-3643 許可制代幣／RWA 上架（CEX 充提）

> 銜接：一般上架清單見 [token-listing-onchain-checklist.md](./token-listing-onchain-checklist.md)、凍結／黑名單見 [stablecoin-blacklist-freeze.md](./stablecoin-blacklist-freeze.md)。今日唔重複 decimals／proxy，而係講 **permissioned token**：轉帳要過身份同合規檢查，發行方仲可以凍結、強制轉帳、回收地址。

## 人話（從你熟的 CEX）

普通 ERC-20 好似「誰有私鑰誰就能轉」。ERC-3643（T-REX，常被用喺證券型／RWA 代幣）唔係：每一次 `transfer` 合約都會先問兩件事——收款地址喺呢隻幣嘅 **Identity Registry** 入面有冇過關（`isVerified()`），同埋今次轉帳符唔符合呢隻幣掛住嘅 **Compliance** 規則（`canTransfer()`，例如國家、合格投資者、持有人數上限、鎖定期）。任一項唔過，交易直接 revert，鏈上唔會有 `Transfer`。

身份唔等於錢包地址。投資者有一個 **OnchainID**（身份合約），可以綁多個錢包；鏈上只放證明（claims）嘅哈希同引用，KYC 個資留喺驗證機構。發行方可以撤銷 claim，之後呢個地址就轉唔到、收唔到。發行方／transfer agent 仲有 `setAddressFrozen`、`freezePartialTokens`、`forcedTransfer`、`recoveryAddress`，同 `pause`。對托管嚟講，呢啲特權等於「發行方可以喺你唔簽名嘅情況下，凍結甚至搬走熱錢包裡嘅幣」——上架前要當托管風險，唔係普通 ERC-20。

## 面試短答

1. **充值地址先要被呢隻幣登記同驗證。** 交易所嘅充值／熱／冷錢包都要喺該 token 嘅 Identity Registry 入冊，並且具備發行方信任嘅 claims。未驗證，用戶轉入會 revert，佢會以為「充值失敗」，你哋 scanner 亦唔會見到 `Transfer`。唔好假設「開咗一個普通充值地址就收得到」。
2. **出金一定要預檢。** 廣播前用只讀接口查 `isVerified(to)` 同 `canTransfer(from, to, amount)`。用戶出金地址未過 KYC、claim 過期、國家唔再允許、或觸及持有人上限，tx 會失敗。規則由發行方嘅 compliance 合約決定，**owner 可以換成新規則，下一筆轉帳即時生效**。
3. **監控發行方特權，唔只掃 `Transfer`。** 訂閱 `AddressFrozen`／`TokensFrozen`、`Paused`／`Unpaused`、`RecoverySuccess`、Identity／Compliance 合約更換。`forcedTransfer` 仍然會出 `Transfer`（若解凍仲可能有 `TokensUnfrozen`），但 **唔係你哋簽嘅出金**；`recoveryAddress` 可以把整個餘額（連凍結部分）從舊地址搬去新地址，並把舊地址移出 registry。熱錢包被 freeze／被強制轉走，要當 P0 托管事件，唔好當普通出金或用戶提現。
4. **強制轉帳同回收跳過部分規則，但收款方仍必須已驗證。** 標準寫明 agent 先可以做；`forcedTransfer` 唔走 offering 嘅 transfer rules，`recoveryAddress` 要求新錢包已經係該 OnchainID 嘅 management key。社交工程若令發行方「幫你回收」充值地址，幣會離開托管。
5. **歸集同內部調撥同樣要過關。** 充值地址 → 熱錢包 → 冷錢包，三個地址都要 verified，否則 sweep 會 revert。Compliance 改規則或撤銷 claim 之後，以前 succeess 嘅路徑都可以突然失敗。

## 常見追問（連答案）

**Q1：scanner 只訂閱 `Transfer`，夠唔夠為 RWA 入帳？**  
A：成功轉入仍然靠 `Transfer` 入帳（revert 嘅根本冇 event）。但唔夠：你要同時聽凍結、暫停、回收同 registry 變更。地址被 freeze 後，帳本上仲有餘額，但你轉唔走；被 `forcedTransfer`／`recoveryAddress` 搬走就唔係用戶提現。另外，未驗證地址根本收唔到幣，唔好把「用戶聲稱已轉」當成鏈上已入。

**Q2：用戶出金到一個從未做過呢隻幣 KYC 嘅地址，會點？**  
A：`transfer` 會 revert。正確係預檢失敗就拒單並提示「收款地址未通過該資產身份驗證」，唔好重試、亦唔好當鏈上已出帳。即使用戶以前收過呢隻幣，claim 被撤或規則更新後都可以再失敗，每次出金都要預檢。

**Q3：呢種幣同 USDT／USDC 黑名單有咩分別？托管點看？**  
A：穩定幣黑名單通常係發行方凍結你轉帳；ERC-3643 再加一層「誰可以持有」，而且標準要求有 recovery，agent 可以 `forcedTransfer`。CEX 上架要同發行方書面搞清楚：邊個係 agent、凍結／強制轉帳嘅通知同申訴、充提地址點樣批量登記、規則變更點通知。唔好只做普通 ERC-20 checklist。

## 小練習（題目 + 參考答案）

**題目：** 產品要上架一隻 ERC-3643 房地產代幣。後台用現有 ERC-20 流程：生成普通充值地址、只掃 `Transfer`、歸集到未登記嘅熱錢包、用戶出金直接 `transfer`。上線第一日，有用戶轉入失敗、有一筆歸集 revert、另有一筆熱錢包餘額突然少咗但冇出金單。有咩問題？點改？

**參考答案：**

- **充值地址未入 Identity Registry**：用戶 `transfer` revert，唔會有 `Transfer`。上架前要同發行方把充值／熱／冷（同歸集路徑上每個地址）登記並取得所需 claims；演練一筆真實轉入必須成功入帳。
- **歸集目標未驗證**：即使充值地址已 verified，掃去未登記熱錢包一樣會失敗。內部調撥同樣要預檢 `canTransfer`。
- **熱錢包少幣好可能係 `forcedTransfer` 或 `recoveryAddress`**：要對鏈上 agent 交易同 `RecoverySuccess`，當托管事件告警，唔好當用戶提現或對帳誤差就抹平。同時訂閱凍結／暫停，凍結後停止出金同歸集。
- **出金要預檢**：對 `isVerified(to)`＋`canTransfer`，失敗就拒單。Compliance／claim 變更要當配置事件，必要時暫停呢隻幣充提。
