# 穩定幣發行方黑名單凍結（USDT／USDC）對交易所充提嘅影響

## 人話

USDT、USDC 係中心化發行方嘅合約，發行方可以將某個地址拉入黑名單。被拉黑嘅地址啲幣仲喺 `balanceOf` 度見到，但轉唔走；Tether 仲可以 `destroyBlackFunds` 直接燒走。對交易所即係：熱錢包／充值地址一被拉黑，錢會卡死；出金打去黑名單地址，會失敗或者打咗出唔返。

## 面試短答

USDC（FiatToken）用 `notBlacklisted` 檢查 `msg.sender`、`from` 同 `to`，所以發送方或收款方喺黑名單都會 revert；USDT 舊合約主要檢查發送方（`isBlackListed[msg.sender]`／`transferFrom` 嘅 `_from`），所以打入黑名單地址可以成功但收咗之後轉唔走。交易所要做三樣：(1) 出金簽名前 `eth_call` `isBlacklisted`／`getBlackListStatus`（加 OFAC／KYT 制裁名單）預檢；(2) 監聽自己地址相關嘅 `AddedBlackList`／`Blacklisted`／`DestroyedBlackFunds` 事件，命中即 P0；(3) 資金分散喺多個地址，唔好一個熱錢包放晒，減少單點被凍嘅損失。

## 常見追問（連答案）

**Q：`destroyBlackFunds` 對帳點睇？**  
A：佢會 emit `DestroyedBlackFunds` 但冇 `Transfer` 事件，靠 Transfer 流水對帳會漏，要用 `balanceOf` 快照對帳先捉到差額，再按事件解釋。

**Q：用戶充值地址因為收咗黑錢被拉黑點算？**  
A：該地址資金標記 `frozen`，唔再歸集；用戶帳戶按風控／法務流程凍結；同發行方／執法溝通解凍。帳本上清楚區分「交易所自有資金被凍」同「用戶資金被凍」，前者係損失撥備，後者要睇法務決定。

**Q：點解唔可以淨係靠鏈上 revert 擋？**  
A：USDC revert 都要付 gas、用咗 nonce、出金單變失敗要退款；USDT 根本唔會 revert，錢直接卡喺對方。預檢平過事後處理。

**Q：預檢有冇 race？**  
A：有，簽名到上鏈之間地址可能先被拉黑，所以預檢係減少而唔係消滅；上鏈後仍要睇 receipt status 同事件。

## 小練習（題目後即附參考答案）

**題：** USDC 出金已經簽咗廣播，receipt `status=0`（revert，原因係收款地址喺黑名單）。點處理？

**參考答案：**  
(1) nonce 已用、gas 已扣，唔使填洞，唔好自動重試（重試只會再 revert）；(2) 出金單標記 `failed`，用戶凍結款退返可用餘額（手續費按政策退／唔退），gas 記成本；(3) 該地址加入內部黑名單／地址簿標記；(4) 根因：預檢冇做或者冇覆蓋 USDC，補上 `isBlacklisted` `eth_call` + 制裁名單篩查；(5) 如果係 USDT 呢種唔 revert 嘅情況，錢已經去咗，只能客服 + 發行方流程，所以預檢更重要。
