# TON Jetton（例如 USDT on TON）充值掃描

## 人話（從你熟的 CEX）

TON 上每個用戶、每種 **jetton** 都有獨立嘅 **jetton wallet 合約**（似 Solana **ATA**）。轉帳唔係一筆簡單 transfer，而係 **異步 message 鏈**：

1. 發送方 **jetton wallet** 收到 `transfer`（op **`0x0f8a7ea5`**）
2. 發 **`internal_transfer`**（op **`0x178d4519`**）去接收方 jetton wallet
3. 接收方 jetton wallet 加餘額；若 **`forward_ton_amount > 0`**，再發 **`transfer_notification`**（op **`0x7362d09c`**）俾 **owner**（交易所地址）
4. 多餘 TON 以 **`excesses`**（op **`0xd53276db`**）退返

CEX 入帳唔可以只睇「有一筆 notification」，要信 **真 jetton wallet** 同成條 trace。

## 入帳要點

### (a) 真 jetton wallet，唔信假 notification

用 **jetton master** 的 **`get_wallet_address(我方 owner)`** 計出「我方真 jetton wallet」。只信：

- 由呢個地址發出嘅 **`transfer_notification`**，或
- 直接睇呢個 jetton wallet 收到嘅 **`internal_transfer`**

假 jetton 可以自己發 **`transfer_notification`** 聲稱係 USDT；只睇 notification 內容會被騙。

### (b) Master 白名單

同 EVM 用 **`chainId + contract`** 做 key 一樣；jetton **master 地址**入白名單，唔好淨靠 symbol／metadata。

### (c) 多筆 transaction，追成條 trace

一筆轉帳可能跨多筆 **transaction**。第一筆成功**唔代表** jetton 已入到你個 wallet；要追 **接收方 jetton wallet** 嗰筆 **compute / action phase** 成功。

### (d) 共用充值地址 + comment（memo）

`forward_payload` 開頭 **32-bit `0`** 表示 **text comment**，用 comment 做用戶識別。冇 comment／錯 comment → **無主充值**流程。

### (e) 冪等 key

用 **交易 hash + logical time（lt）** 或 **message hash**；**唔好用 comment** 做唯一 id（可重放／碰撞風險）。

### (f) 最終性

**Shard block** 被 **masterchain block** 引用後當最終（BFT，冇長 reorg）；以 **masterchain seqno** 做掃描游標。

## 地址格式

同一 **raw 地址**（`workchain:hex`）有多種 user-friendly 形式：**bounceable（EQ…）**、**non-bounceable（UQ…）** 等。DB **一律存 raw** 做比較。出金去**未初始化**錢包要用 **non-bounceable**，否則 **bounce** 退返。

## 面試短答

- 入帳：**master 白名單** + **`get_wallet_address`** 鎖定我方 jetton wallet + **internal_transfer / notification 溯源**。
- 共用地址靠 **comment**；冪等靠 **tx hash + lt**。
- 游標：**masterchain seqno**；最終性 = masterchain 引用 shard。
- 兜底：即使 **`forward_ton_amount = 0`** 收唔到 notification，都要監控我方 jetton wallet 的 **internal_transfer**。

## 常見追問（連答案）

**Q：點解 `transfer_notification` 可能收唔到？**  
A：發送方設 **`forward_ton_amount = 0`** 就唔會發 notification 俾 owner。所以要**直接監控我方 jetton wallet 的 internal_transfer** 兜底，唔可以只靠 notification。

**Q：bounce 了點處理？**  
A：出金失敗退款會以 **bounced message** 返嚟。要匹配原出金單、標記失敗並**解凍**用戶餘額，**唔好當新充值**。

**Q：收到 notification 話有 500 USDT，但我方 jetton wallet 餘額冇變？**  
A：個 notification **唔係由我方真 jetton wallet**（`get_wallet_address` 計出）發出 → 假 jetton／偽造，**唔入帳**，記錄做風控樣本。

## 小練習

**題：** 收到一筆 `transfer_notification` 話有 500 USDT 入咗，用戶 `comment=8812`，但我方 USDT jetton wallet 餘額冇變。點判斷？

**參考答案：**  
Notification 發送方地址 ≠ `get_wallet_address(USDT_master, 我方 owner)` → **偽造／假 jetton**，唔入帳；查 master 白名單、記錄風控樣本，必要時提示用戶勿信第三方「已到账」截圖。
