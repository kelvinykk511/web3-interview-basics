# Solana 出金：blockhash 過期重簽 + priority fee

## 人話
Solana 冇 EVM 嗰種 nonce。每單 tx 要簽一個「最近 blockhash」，大約 150 個 block（約 60–90 秒）之後就過期，過期咗嗰張 tx 永遠唔會上鏈。
所以「卡單」唔係好似 EVM 咁加 gas 替換同一個 nonce，而係：等佢確定過期 → 先至用新 blockhash 重簽。

## 面試短答
- 簽名嗰陣記低 `lastValidBlockHeight`（`getLatestBlockhash` 會一齊返）。
- 廣播後用 signature 查 `getSignatureStatuses`（要開 `searchTransactionHistory`），睇 confirmed / finalized。
- 未上鏈 + 當前 block height **未超過** `lastValidBlockHeight` → 只可以**重發同一張已簽 tx**（同一 signature，網絡會去重，唔會雙花）。
- 當前 block height **已超過** `lastValidBlockHeight`，而且查唔到 signature → 舊單已死，先可以用新 blockhash 重簽，出金單記錄新 signature（舊 signature 留底做審計）。
- 想離線簽 / 慢簽（例如冷錢包、多簽審批），用 **durable nonce**：nonce account + 第一條指令 `AdvanceNonceAccount`，tx 唔會因時間過期；同一個 nonce 只可以成功一次，天然防重複出金。

## Priority fee
- base fee：每個簽名 5000 lamports（固定）。
- priority fee = `compute unit price`（micro-lamports / CU）× `compute unit limit`，用 ComputeBudget 指令設定。
- 做法：先 simulate 攞實際 CU 用量 → limit 設用量 × 1.1–1.2；price 參考 `getRecentPrioritizationFees`（睇相關帳戶，例如 token mint / 熱錢包）取分位數，再設上限。
- limit 設太大 = 白白多付（price × limit）；太細 = tx 執行失敗但照收費。
- 網絡擠：重簽嗰陣可以順手加 price（因為反正要新 blockhash）。

## 常見追問（連答案）
1. **點解唔可以未過期就重簽？**
   答：新舊兩張 tx signature 唔同，兩張都有效，兩張都可能上鏈 → 雙倍出金。一定要等舊單確定過期（或用 durable nonce 令兩張搶同一 nonce）。
2. **`sendTransaction` 自帶 retry 夠唔夠？**
   答：唔夠可靠。常見做法 `maxRetries: 0`，由自己服務每 2 秒重發同一張 tx，直到 confirmed 或過期；咁先知道幾時可以安全重簽。
3. **blockhash 用咩 commitment 攞？**
   答：一般用 `confirmed`。用 `processed` 可能攞到少數派分叉嘅 blockhash，tx 永遠唔落，白等到過期。
4. **confirmed 就扣帳完成？**
   答：出金狀態可以 confirmed 先標「已廣播成功」，最終完成用 finalized（大額尤其）；Solana confirmed 被 rollback 好罕見，但對帳以 finalized 為準。
5. **tx 上咗鏈但失敗（例如 CU 唔夠）？**
   答：Solana 失敗 tx 照收 fee、有 signature、狀態有 `err`。出金單要標失敗、手續費入成本，再開新 tx（新 blockhash），唔好當成功。

## 小練習（附參考答案）
**題：** 出金服務 10:00:00 簽咗一張 USDC 轉帳，`lastValidBlockHeight = 1000`。10:01:30 查 signature 返 null，當前 block height = 990。你應該點？

**參考答案：** 未過期（990 ≤ 1000），唔可以重簽。繼續重發**同一張** tx，等到 confirmed 或 block height > 1000。過咗 1000 再查一次 signature（開 history search）仍然 null，先標舊單 expired，用新 blockhash（可加 priority fee）重簽，記新 signature。

**題：** 冷錢包審批流程要 30 分鐘，點簽 Solana 出金？
**參考答案：** 用 durable nonce account。tx 用 nonce 值代替 blockhash，第一條指令 advance nonce，就唔會 90 秒過期；而且同一 nonce 只可以用一次，就算重複提交亦唔會雙出。
