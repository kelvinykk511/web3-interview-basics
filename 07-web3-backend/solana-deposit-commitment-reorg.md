# Solana 充值：commitment 層級、reorg 與 CEX 入帳

> 銜接：[充值 Reorg／確認數](./deposit-reorg-confirmation-depth.md) 講 EVM 確認數；[L2 充值最終性](./l2-deposit-finality.md) 講 L2 soft/safe/finalized；[Solana 出金 blockhash](./solana-withdrawal-blockhash-priority-fee.md) 講出金側 commitment。今日講 **Solana 冇「12 個 block」——用 processed/confirmed/finalized 綁入帳狀態機**。

## 人話

CEX 充值本質係：**鏈上見到一筆轉帳 → 內部帳本加錢**。EVM 習慣講「等 N 個確認」；Solana 用 **commitment level** 描述「呢個 slot／signature 有幾穩」。

可以咁類比支付渠道：

| Solana commitment | 大概意思 | CEX 類比 |
|-------------------|----------|----------|
| **processed** | 某個 leader 已經排入 block，但仲可能被丟棄 | 「渠道回調 pending」——快，但可能撤銷 |
| **confirmed** | 超級大戶（~2/3 stake）已投票，極少 rollback | 「已清算、極少 chargeback」——多數小中額可入帳 |
| **finalized** | 已 rooted，協議層視為不可逆 | 「對帳／大額放貨」——財務認定 |

**Reorg／rollback** 喺 Solana 唔似 PoW 咁「深鏈取代」咁常見，但 **processed** 層仍然可能「slot 被 skip、tx 從未上 canonical chain」。如果你喺 processed 就 credited，就要準備 **冲正（reverse ledger entry）**：用 `signature`（同 `slot` 做審計）標記作廢，唔好淨係改 pending 就算。

典型 CEX 做法：

- UI **早顯示**：`processed` 或 `confirmed` 就顯示「確認中／已到帳（待解鎖）」。
- **可用餘額（可交易／可提現）**：多數等小額 `confirmed`、大額 `finalized`（分 tier）。
- **財務對帳、日結**：以 `finalized` 為準，同 EVM 用 `finalized` block tag 一個思路。

查鏈時 **`getTransaction` / `getSignatureStatuses` 嘅 `commitment` 參數必須同入帳政策一致**——唔好 processed 輪詢見到就入帳，卻又用 finalized 話「已對帳完成」；亦唔好 A 節點 processed、B 節點 finalized 混用同一筆狀態推進。

**RPC lag／多節點**：兩個 healthy 節點喺 **confirmed** 邊界可能短暫唔一致（一邊已見、一邊未見）。入帳應 **sticky 同一節點** 或 **quorum（例如 2/3 節點一致）**，推進到 credited 前再 **re-check** 一次（尤其大額升 finalized）。

**同 EVM 對照（面試一句）**：EVM 充值常講 `confirmations = head - block + 1` 或 `safe`/`finalized` tag；Solana 用 **同一個 RPC 參數 commitment** 貫穿「掃描、查 receipt、入帳門檻」。L2 筆記嘅 **soft / safe / finalized** 係「sequencer vs L1 batch」層次；Solana 三級係 **投票深度**，概念類似「愈深愈唔使回滾帳本」。

**冪等鍵**：Solana 原生／SPL 轉帳一般 **`signature` 全局唯一**；若一 signature 內有多個 transfer 指令影響同一充值地址，要用 **`signature + instruction index`**（或解析後 `(mint, destination, amount)` 做子鍵），防 partial 重放。

## 面試短答

- Solana 充值唔背「12 block」，背 **processed → confirmed → finalized** 同產品綁定：顯示 pending、可 spend、credited 對帳各用邊一級。
- **processed** 可 drop；**confirmed** rollback 好罕；**finalized** 大額入帳同 reconciliation。
- Reorg／skip：若已 credited 但 signature 後來唔喺 finalized 歷史 → **冲正**（idempotent by signature），狀態 `reversed` / `reorged`。
- RPC：`getSignatureStatuses(sig, { searchTransactionHistory: true })` 嘅 commitment 同 credit policy 一致；多節點用 quorum 或 sticky + 推進前複查。
- 對照 EVM：確認數／`safe`/`finalized`；對照 L2：soft 快但弱、finalized 慢但強——Solana finalized 通常快過等 L1 L2 batch finalized。

## 常見追問（連答案）

1. **點解唔可以全部用 processed 入帳？**
   答：processed 只代表某 leader 排過，slot 可能 skip、tx 從未進 canonical chain。用戶已經用「可用餘額」買幣／提現之後先發現從未真正上鏈 → 帳本災難。最多用 processed 做 UI「偵測到」或 `seen`，credited 至少要 confirmed（小額）或 finalized（大額）。

2. **confirmed 已 credited，會唔會被 rollback？**
   答：協議上極罕，但 CEX 設計唔靠「從未發生過」。大額用 finalized；若監控到 cluster 異常／深度不一致，凍結出金。真係要回滾：用 signature 做冲正，已花出去嘅餘額走風控／異常流程，同 EVM「credited 後 reorg」同一類 incident。

3. **`getTransaction` 同 `getSignatureStatuses` 用咩 commitment？**
   答：同配置寫死。例如：掃描用 `confirmed`；推進到「可提現」查 `finalized` 狀態為 `finalized` 先升 tier。唔好用 `processed` poll 到 `confirmationStatus: processed` 就直接加可用餘額，卻對帳用 finalized——語意混亂兼易踩 lag。

4. **同 EVM 12 確認點比？**
   答：EVM 確認數係「head 往前幾個 block」嘅概率安全；Solana confirmed 係 supermajority 投票、finalized 係 rooted。面試講：**唔係換算 block 數，而係換算「產品接受嘅 rollback 概率」**——小額 confirmed、大額 finalized、對帳 finalized。

5. **同 L2 soft / safe / finalized 有咩似？**
   答：L2 soft 快但 sequencer 風險；safe/finalized 綁 L1。Solana processed 似「好快但可撤」；finalized 似「可以當財務定案」。Solana 通常幾十秒到兩三分鐘級 finalized，唔使等 L1 batch 嗰種十幾分鐘（除非跨鏈橋資產另講）。

## 小練習（附參考答案）

**題：** 政策：小於 1 萬 USDC 等 `confirmed` 入可用；≥1 萬 等 `finalized`。Scanner 用 RPC-A `confirmed` 見 signature S，slot 250_000_001，已推進 `credited`。對帳 job 用 RPC-B `finalized` 查 S 返 null，但 RPC-A `confirmed` 仍話存在。你第一步做咩？

**參考答案：** 唔好當已對帳完成。凍結該筆相關出金／大額操作；用 sticky 或第三節點 quorum 再查 S 喺 `confirmed` 同 `finalized` 嘅狀態同 slot。若多數節點 finalized 都 null → 標 `reorged`／冲正（signature S），游標重掃 slot 附近；若只係 RPC-B lag → 等 finalized 追上再解凍。記錄兩節點不一致做 RPC 告警。

**題：** 一張 tx 入面兩條 SPL Transfer 都入同一用戶充值 ATA，共用 signature `5abc...`。冪等鍵點設？

**參考答案：** 唔好用 signature  alone 一條帳（會漏第二條）。用 `(signature, instruction_index)` 或解析後 `(signature, destination, amount, mint)` 各一條 deposit row；重播 indexer 時 unique constraint 防雙加。冲正時按 row 逐條 reverse，或整筆 tx 標 invalid 再逐條冲正。
