# USDT／USDC Blacklist：對充值、歸集、出金嘅影響

## 人話（從你熟的 CEX）

CEX 自己可以凍結用戶帳戶；鏈上嘅穩定幣**發行方**（Tether、Circle）都可以凍結任何地址——包括你交易所嘅充值地址同熱錢包。被凍結之後，個地址嘅幣喺合約層面郁唔到，你內部帳本話有都冇用。

關鍵細節：**兩隻幣規則唔同**（以 Ethereum 主網合約為例）：

| | USDT（TetherToken） | USDC（FiatToken） |
|---|---|---|
| 查詢 | `isBlackListed(addr)` | `isBlacklisted(addr)` |
| 被凍地址轉出 | revert | revert |
| **轉入**被凍地址 | **會成功**（錢入咗但郁唔到） | **revert**（收款方都檢查） |
| 事件 | `AddedBlackList` / `RemovedBlackList` | `Blacklisted` / `UnBlacklisted` |
| 銷毀 | `destroyBlackFunds` 可清零被凍地址餘額 | 由合約管理員操作，視版本 |

## 面試短答

- **充值地址被凍**：歸集（sweep）會 revert，燒 gas 又搬唔走。歸集前先 `eth_call` 查 blacklist／模擬 transfer；命中就標 `FROZEN`，**唔好重試／RBF**，轉風控同合規。
- **熱錢包被凍**：最嚴重——該幣所有出金即刻失敗。防線：分散多個熱錢包、限額、監聽 blacklist 事件（命中自家地址即 P0 告警、暫停該錢包出金）。
- **出金去被凍用戶地址**：USDC 會 revert（浪費 gas＋失敗單）；USDT 會成功但用戶收到郁唔到嘅錢 → 客訴。兩者都應**出金前預檢**目標地址，命中就拒絕。
- **對帳**：被凍餘額唔應算入可用資產／儲備證明嘅「可動用」部分；USDT 舊合約 `destroyBlackFunds` 只 emit `DestroyedBlackFunds`、**冇 `Transfer` 事件**，淨靠 Transfer 對帳會漏 → 要定期 `balanceOf` 核對。

## 常見追問（連答案）

**Q：點解 blacklist revert 要當「永久失敗」而唔係「可重試」？**  
A：gas 唔夠、nonce 卡住係暫時問題，加 gas 會好；blacklist 係合約狀態，你加幾多 gas 都會 revert，重試只係燒錢兼塞住 nonce 佇列。錯誤分類要按 revert 原因（或預檢結果）走唔同分支：`FROZEN` → 人工／合規單。

**Q：用戶由一個「之後先被凍」嘅地址充值俾你，點算？**  
A：轉帳當時對方未被凍，合約層面成功，錢已喺你充值地址，**你個地址冇被凍**，歸集正常。問題喺合規：KYT（鏈上資金風險掃描）可能標記來源有問題，要按合規政策 hold 住或者上報，呢個係業務／合規決定，唔係合約限制。

**Q：點樣及早知道自家地址被凍？**  
A：訂閱／掃描兩個合約嘅 blacklist 事件，同自家地址庫（充值地址＋熱錢包＋冷錢包）做 match；加上定時批量 `isBlackListed`／`isBlacklisted`（可用 multicall）做兜底，防事件漏掃。

## 小練習

**題：** 夜晚歸集任務報錯：某 USDC 充值地址 sweep 連續 revert，系統自動 RBF 重試咗 6 次。同一時間有用戶申請提 USDT 去一個地址，風控 API 查到該地址已被 Tether blacklist。你點處理兩件事，同埋點改系統？

**參考答案：**  
(1) 歸集：即刻停止重試（6 次都係燒 gas）。`eth_call isBlacklisted(addr)` 確認係 USDC 凍結 → 標該地址 USDC 餘額為 `FROZEN`，由可用資產剔除，開合規／風控單（查邊筆充值引起、對應用戶帳戶要唔要凍）。(2) USDT 出金：雖然 USDT 轉入被凍地址會成功，但用戶會收到郁唔到嘅錢 → 拒絕出金並通知用戶／客服，唔好因為「鏈上會成功」就放行。系統改善：歸集同出金前都加 blacklist 預檢（`eth_call`，必要時 multicall 批量）；revert 錯誤分類，blacklist 走永久失敗分支唔 RBF；監聽 `Blacklisted`／`AddedBlackList` 事件對自家地址庫，熱錢包命中即 P0 停該錢包出金；定時 `balanceOf` 對帳兜住冇 Transfer 事件嘅銷毀。
