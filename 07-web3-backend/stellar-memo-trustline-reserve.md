# Stellar 充提：memo、trustline、base reserve、muxed account

## 人話
Stellar 好似 XRPL：交易所通常用**一個熱錢包地址 + memo** 分用戶。但有三樣嘢 EVM 冇：
1. **帳戶要「開戶」**：新地址要收到一筆 `create_account`（帶最少 XLM）先存在，直接 `payment` 去未開戶地址會失敗。
2. **最低餘額（base reserve）**：每個帳戶要鎖住一筆 XLM（2 × base reserve），每多一個「subentry」（trustline、offer、signer、data）再加 0.5 XLM。呢啲錢唔可以用。
3. **Trustline**：想收 USDC 之類資產，收款帳戶先要 `change_trust` 開 trustline；資產身份 = **asset code + issuer 地址**，唔係淨係 "USDC"。

## 面試短答
- 充值：掃 Horizon/RPC 嘅 payments，用 memo（MEMO_ID / MEMO_TEXT）對用戶；資產用 `code + issuer` 白名單；只認 `successful=true` 嘅交易；ledger 一 close 就 final（SCP 冇 reorg），唔使等多個確認。
- 冇 memo / memo 錯 → 入「待認領」，人工流程，唔好自動入帳。
- 出金：先查目標帳戶存唔存在、有冇對應 trustline；未開戶 + 出 XLM ≥ 最低開戶額 → 用 `create_account`；出 token 但對方冇 trustline → 直接拒絕（返明確錯誤），唔好發一筆注定失敗嘅交易。
- 帳戶有 **sequence number**（似 nonce），熱錢包並發要統一分配；交易有 `timebounds`，過期 = 確定作廢，可以安全用新 seq 重建。
- 熱錢包可用餘額 = 總 XLM − 最低餘額 − 預留手續費，風控要按呢個數算。

## 常見追問（連答案）
**Q1：Muxed account（M 開頭地址）係咩？點處理？**
答：SEP-23，將「G 地址 + 64-bit ID」編成一個 M 地址，等於把 memo 塞入地址。充值時要解出底層 G 地址同 ID，ID 當用戶標識；如果交易所唔支援，地址校驗要明確拒絕 M 地址，唔好當普通地址。出金到 M 地址時，唔好再另外要用戶填 memo。

**Q2：點解「交易成功」都可能冇真正收到錢？**
答：要逐個 operation 睇——一筆交易可以有多個 operation，只入帳 destination = 我哋、asset 喺白名單、類型係 `payment` / `path_payment_strict_*` 嗰啲。Path payment 要用**實際到帳金額**（destination amount），唔好用 source 金額。仲有 claimable balance：別人可以 `create_claimable_balance` 畀你，唔 claim 就唔算到帳。

**Q3：Issuer 可以凍結你嘅 token 嗎？**
答：可以。資產可設 `AUTH_REQUIRED`、`AUTH_REVOCABLE`、`clawback`。上幣前要查 issuer flags；有 clawback 嘅資產，對帳要監控 clawback operation，似 USDT 黑名單嘅處理。

**Q4：交易提交後超時冇回應點算？**
答：用 tx hash 查；未查到就等到 `timebounds.maxTime` 過咗 + 個 ledger close，再查一次仍冇 → 確定作廢，先用新 sequence 重建。超時前唔好用新 seq 重發同一筆出金，會雙花。

## 小練習（附答案）
**題：** 用戶要提 100 USDC 去一個全新 Stellar 地址 `GABC...`，你個系統應該點做？

**參考答案：**
1. 查 `GABC...` → 帳戶不存在。
2. 出 token 需要對方帳戶存在 + 有 USDC（正確 issuer）trustline，兩樣都冇 → 拒絕出金，提示「目標帳戶未激活／未開 USDC trustline」。
3. 唔好自作主張幫佢 `create_account` 再等佢開 trustline（要佢自己簽 `change_trust`，交易所做唔到）。
4. 可以選擇改用 claimable balance 送畀佢（佢開 trustline 後自己 claim），但要產品同意，並寫清楚「未 claim 前資金喺鏈上鎖住」。
