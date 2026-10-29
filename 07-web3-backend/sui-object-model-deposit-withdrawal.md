# Sui：Object 模型下嘅充值 / 出金

## 人話
Sui 唔係 EVM 嘅「帳戶 → 餘額數字」，而係 **object（物件）模型**：你嘅 SUI 同 USDC 係一粒粒 `Coin<T>` object，每粒有自己 ID、version、餘額。好似你銀包入面唔係「總數 500 蚊」，而係「一張 100、兩張 200」咁。

對 CEX 嚟講要識三樣嘢：
1. 錢係一粒粒 coin，**收得多就碎**，要 merge（合併）。
2. 同一粒 owned object **唔可以同時畀兩筆交易用**，搞錯會鎖死到 epoch 完結（約 24 小時）。
3. 入帳睇交易 effects 嘅 **balanceChanges**，唔好自己逐個 object 計。

## 面試短答
> Sui 係 object 模型，餘額由多個 `Coin<T>` object 組成。充值用每個用戶獨立地址，按 checkpoint 掃交易，用 transaction effects 嘅 `status=success` 同 `balanceChanges`（owner=我哋地址、coinType 精確匹配白名單）入帳，冪等鍵用 `digest + coinType + owner`。
> 出金嘅坑係並發：owned object 被兩筆並發交易同時用會 equivocation，object 鎖到 epoch 尾，所以熱錢包要為每個 worker 預分配獨立 gas coin 同資金 coin（或者一個 worker 串行），定期 merge 碎 coin。Sui 有 BFT 即時 finality，交易進 checkpoint 就 final，唔使等確認數。

## 重點細節
- **地址**：`0x` + 64 hex（32 bytes），出金要 normalize（補零、小寫）再校驗。
- **Coin type 用完整 type tag**：例如 SUI 係 `0x2::sui::SUI`，其他 token 係 `<package>::<module>::<STRUCT>`。同名唔同 package 嘅假幣好多 → 白名單要精確到完整 type。
- **decimals**：SUI 係 9；其他幣查 `CoinMetadata`，按 coin type 存，唔好假設。
- **Gas**：用一粒 SUI coin 付 gas（gas coin），未用完嘅退返；有 storage fee 同 storage rebate（刪 object 退返部分）。
- **Finality**：Sui 交易確定後就唔會 reorg；以 checkpoint 做掃鏈游標（checkpoint sequence number 單調遞增），斷點續掃好處理。
- **Coin 碎片**：大量小額充值 → 地址有幾千粒細 coin；一筆交易輸入 object 有上限，歸集要分批 `mergeCoins` 再 `transferObjects`。
- **PTB（Programmable Transaction Block）**：一筆交易可以 split + merge + 轉畀多個人，批量出金好方便，但任何一步失敗成筆 revert。

## 常見追問（連答案）
**Q1：咩係 equivocation？點樣會發生？**
A：兩筆唔同交易同時用同一粒 owned object（同一 version），送去唔同 validator，各自拎到部分簽名但都唔夠 2/3，object 就被鎖，直到 epoch 結束先解。典型成因：兩個出金 worker 同時用同一粒 gas coin，或者 RPC 超時之後用同一粒 coin 重新組另一筆交易。
防法：每個 worker 獨立 gas coin pool；超時重試要重送**同一筆已簽好嘅交易**（同 digest），唔好重新組；或者用 shared object 設計。

**Q2：入帳點解用 balanceChanges，唔用 object 變化？**
A：一筆交易可能 split/merge/轉多粒 coin，自己由 object created/mutated 計好易錯；`balanceChanges` 已經按 owner + coinType 算好淨變化。仍要檢查 `effects.status == success`，失敗交易只扣 gas 冇轉帳。

**Q3：出金 RPC 超時，唔知成唔成功，點做？**
A：簽名後交易 digest 已確定，先存低 digest，再用 `getTransactionBlock(digest)` 查。查唔到就重送**同一份簽好嘅 bytes**（冪等，因為同一筆交易），唔好用新 coin 重新組交易，否則可能雙付或 equivocation。

**Q4：同 EVM nonce 有咩分別？**
A：EVM 用帳戶 nonce 排隊，一個帳戶串行；Sui 冇帳戶 nonce，排隊單位係 object version。好處：用唔同 coin 嘅交易可以並行；壞處：要自己管 coin 分配，用重同一粒就衝突。

**Q5：充值地址收到一種白名單以外嘅 coin 點算？**
A：唔入帳、記錄做異常（可能係垃圾空投/釣魚 token），唔好自動歸集（歸集要 gas，仲可能係惡意 package）。

## 小練習（題目 + 參考答案）
**題目：** 你哋熱錢包有 4 個出金 worker 共用一個地址，最近間中有出金卡 24 小時，交易查唔到成功又查唔到失敗。點排查同改？

**參考答案：**
1. 懷疑 equivocation：查卡住嘅交易用咗邊粒 gas coin / input coin，見到同一 object 同一 version 被兩筆唔同 digest 嘅交易引用 → 證實。
2. 根因：4 個 worker 並發揀「餘額最大嗰粒 coin」做 gas 或資金，撞咗；或者超時後重新組交易用咗同一粒 coin。
3. 改法：
   - 預先 split 出 N 粒 gas coin，每個 worker 綁定自己嘅 gas coin 同資金 coin（coin pool + 租借鎖）。
   - 超時只重送同一份已簽 bytes，用 digest 查結果。
   - 定期 merge 碎 coin、補充 gas coin 餘額，有告警。
4. 被鎖嗰啲出金：等 epoch 結束 object 解鎖，再按 digest 查最終結果，冇成功先重新出，避免雙付。
