# 非標準 ERC-20：fee-on-transfer／USDT／blacklist

## 人話（從你熟的帳本）

CEX 入帳唔可以死信「Transfer 事件嘅 value = 用戶實收」。有啲幣會扣稅（fee-on-transfer）、有啲 `transfer` **唔 return bool**（舊 USDT）、有啲會 **blacklist** 令轉帳 revert 或凍倉。鏈下 ledger 若果用「應收金額」入帳，會同 `balanceOf` 對唔上。

## 面試短答

- **入帳金額以實際到帳為準**：常見做法係 credit 前後 `balanceOf(depositAddr)` 差值，或掃事件後再對一下餘額；唔好只信 calldata／事件 value（對 FoT）。
- **USDT 等非標準**：`transfer`／`approve` 可能無 return；用 OpenZeppelin `SafeERC20`（或等價 wrapper）處理；`approve` 從非 0→非 0 可能 revert（舊坑）。
- **Blacklist／freeze**：合規幣可令提現／歸集失敗；運維要有失敗狀態機同人工／風控單，唔好無限 RBF。
- **Whitelist 上幣**：CEX 上幣 checklist 必測 FoT、rebase、pausable、blacklist、proxy 升級權。

## 常見追問（連答案）

**Q：點解「看 Transfer value 入帳」會錯？**  
A：Fee-on-transfer 時 sender 扣 100、recipient 可能只到 99，事件 value 視實作可能係 100 或 99。用餘額差最穩；若只信事件，要確認該 token 規格。

**Q：Rebase token（彈性供應）又點？**  
A：`balanceOf` 會無 transfer 都變。CEX 多數拒上或用 shares 模型；入帳／對帳要按產品定義，唔好當普通 ERC-20。

**Q：同內部「上帳金額」點對？**  
A：鏈上 credited = 實際入地址數量；用戶看到嘅可用餘額再扣手續費／折扣係 ledger 層。對帳鍵：`chain + contract + txHash + logIndex`（或餘額快照 id）冪等。

## 小練習

**題：** 掃到 `Transfer(from, depositAddr, 1000e6)`（USDT），credit 用戶 1000 USDT 前，你會做邊兩步校驗？

**參考答案：**  
(1) 確認 `contract` 係白名單 USDT（正確 chainId＋地址），唔係假同名幣；(2) 以 `balanceOf(depositAddr)` 增量（或歸集前餘額）核對實收，並檢查該地址／from 唔喺已知異常名單；確認數達標先最終入帳。若 transfer 後餘額無增（blacklist／失敗），唔入帳並打異常單。
