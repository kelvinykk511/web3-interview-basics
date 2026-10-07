# Safe 多簽出金流程（大額／冷錢包）

> 銜接：[Multisig + Timelock（CEX Admin）](./multisig-timelock-admin.md) 講「點解要多簽」；呢篇講「一筆大額出金喺 Safe 上實際點行、後端狀態機點記」。

## 人話
CEX 入面大額出款要「多人覆核」。上鏈之後，冷錢包好多時係一個 **Safe 合約**（M-of-N owners）。流程係：
1. 後端**提案**一筆 Safe 交易（to / value / data / operation / nonce …）。
2. 算出 **safeTxHash**（EIP-712 hash，domain 帶 `chainId` + Safe 地址）。
3. 每個 owner **離線簽** safeTxHash（硬件錢包），簽名收集起嚟（例如 Safe Transaction Service 或者自家簽名服務）。
4. 夠 threshold 之後，**任何一個 EOA（executor）** 呼叫 `execTransaction(...)` 連埋簽名上鏈，executor 付 gas。

關鍵：Safe 有自己嘅 **Safe nonce**（合約內部、嚴格遞增），同 executor EOA 嘅 account nonce **係兩樣嘢**。

## 面試短答
Safe 出金 = 鏈下提案 + 收集 M 個 owner 對 safeTxHash 嘅簽名 + 一個 executor 呼叫 `execTransaction`。後端要記兩層狀態：Safe nonce 隊列（N 未執行，N+1 執行唔到）同 executor 外層 tx（可以卡 gas、要 RBF）。確認結果唔可以只睇 receipt `status=1`，要睇 Safe 事件 `ExecutionSuccess` 定 `ExecutionFailure`。簽名前要獨立 decode 交易內容，特別要拒絕非預期嘅 `operation=1`（delegatecall）。

## 後端狀態機（建議）
`PROPOSED`（已分配 Safe nonce）→ `SIGNING`（k/M）→ `READY`（夠簽）→ `EXECUTING`（外層 txHash，可 RBF）→ `CONFIRMED` ／ `INNER_FAILED` ／ `CANCELLED`（同 nonce 被拒絕交易取代）

- 冪等鍵：業務 withdrawalId ↔ (safeAddress, safeNonce) 一對一；重試唔可以再開新 nonce。
- 外層 tx 卡住：同 [提現 Gas Bump](./withdrawal-gas-bump-ops.md) 一樣用 executor EOA 嘅 nonce 做 RBF；**簽名唔使重簽**（safeTxHash 冇變）。

## 常見追問（連答案）
**Q1：receipt status=1，係咪代表錢出咗？**  
A：唔一定。Safe 有個細節：如果 `safeTxGas` 同 `gasPrice` 都係 0（而家最常見），內層 call 失敗會令成筆 `execTransaction` revert（status=0）；但如果設咗 `safeTxGas > 0`，內層失敗外層照樣成功（status=1），Safe 發 `ExecutionFailure` 事件，**Safe nonce 照樣用咗**。所以要 parse Safe 事件：`ExecutionSuccess(txHash, payment)` 先算出金成功，再加 token `Transfer` log／餘額對帳。

**Q2：已經簽咗一半，發現收款地址錯，點取消？**  
A：Safe 交易冇「刪除」，已簽名喺鏈下仲有效。正確做法係用**同一個 nonce** 提案一筆「拒絕交易」（例如 0 value 轉畀 Safe 自己），收夠簽執行。nonce N 一被用咗，所有其他 nonce N 嘅簽名自動作廢。單係喺 DB 標記取消唔夠，因為有人仲可以拎舊簽名去執行。

**Q3：點解一筆卡住會塞死後面所有出金？**  
A：Safe nonce 嚴格順序。nonce 5 未夠簽／未執行，nonce 6 就執行唔到。所以：大額 Safe 唔好太多筆排隊；卡住要決定「補簽」定「用拒絕交易燒咗 nonce 5」；監控「最老未執行 nonce 嘅等待時間」。

**Q4：`operation` 欄位有乜危險？**  
A：`0`=call、`1`=delegatecall。delegatecall 會用 Safe 自己嘅 storage 執行對方代碼，可以改 owner／改 implementation，等於成個錢包交出去。2025 年 Bybit 冷錢包事件就係簽名人喺被篡改嘅 UI 上簽咗一筆 delegatecall。防線：簽名設備**獨立 decode** to/value/data/operation 並核對 safeTxHash；白名單只容許 call，或者只容許 delegatecall 去官方 MultiSendCallOnly；唔做「blind signing」。

**Q5：executor 用邊個？**  
A：一個專用 EOA，只有 gas 錢、冇資產權限（簽名先有權）。被盜最多損失 gas 錢。要監控佢 ETH 餘額同 pending 狀態，同熱錢包一樣。

## 小練習（附參考答案）
**題：** 一筆 500 ETH 出金，Safe 2/3，Safe nonce=12。兩個 owner 簽完，executor 廣播 `execTransaction`，但 gas 太低卡咗 20 分鐘；期間 ops 再廣播一次 `execTransaction`（新 executor nonce、正常 gas），成功上鏈。舊嗰筆後來點？你個狀態機要點處理？

**參考答案：**
- 新嗰筆成功後，Safe nonce 變 13；舊外層 tx 如果之後被打包，會因為簽名對應 nonce 12 已用咗而 **revert（status=0）**，只係蝕 gas，**唔會重複出金**——Safe nonce 本身就係防重放。
- 但 executor EOA 嘅舊 nonce 仲 pending，會塞住 executor 後續交易：要用同一個 executor nonce 發 0 value 自轉（或者 RBF）清走。
- 狀態機：同一個 (safe, nonce=12) 下記錄多個外層 txHash；任何一個帶 `ExecutionSuccess` 就標 `CONFIRMED`，其他標 `REPLACED/REVERTED`；唔好因為見到一筆 status=0 就判出金失敗、退返用戶餘額（會變雙花）。
