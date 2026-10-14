# BTC UTXO 充值／出金 vs 帳戶模型

## 人話
EVM 係「每個地址有個餘額」。BTC 冇餘額欄位，只有一粒粒「未花費輸出」（UTXO），好似一張張鈔票。
地址「餘額」= 所有屬於佢嘅 UTXO 加埋。要用錢就要成張鈔票花出去，多咗嘅找番零（change output）俾自己。

## 面試短答
- **充值識別**：一張 tx 可以有多個 output，可能幾個都係打俾我哋（甚至唔同用戶）。入帳唯一鍵用 `txid:vout`，唔係 txid。
- **確認數**：0-conf 唔入帳（未確認 tx 可以被 RBF 替換，而家 Bitcoin Core 默認 full-RBF）；通常 1–3 conf 入帳、大額 6 conf 先可提。reorg 時按 `txid:vout` 回滾。
- **出金**：自己做 coin selection 揀 UTXO → 輸出俾用戶 + 找零去自己新嘅 change 地址 → fee = 交易大小（vbytes）× sat/vB。
- **批量出金**：一張 tx 多個 output，比每人一張平好多，交易所常用。
- **歸集 = UTXO 合併**：用戶充值散開成大量細 UTXO，喺 fee 低時合併，否則將來出金 tx 好大好貴。
- **dust**：太細嘅 output（P2WPKH 約 294 sats、legacy 約 546 sats）節點唔轉發；細額充值可能花唔出（花費成本 > 面值）。

## 卡單點救
- **RBF**：原 tx 有標可替換，用同一批 input、更高 fee 重簽 → 舊 tx 作廢。
- **CPFP**：用嗰張卡住 tx 嘅找零 output 再開一張高 fee 子 tx，礦工為咗收子 tx fee 會連父 tx 一齊打包。
- 同 Solana 一樣要防雙出：RBF 用**同一批 input**，所以新舊只可能成功一張；千萬唔好用另一批 UTXO 再出一次。

## 常見追問（連答案）
1. **點解唔可以用 txid 做充值冪等鍵？**
   答：一張 tx 可以有多個 output 打俾我哋唔同地址/用戶，用 txid 會漏單或錯併。要 `txid:vout`。
2. **出金用未確認嘅找零 UTXO 得唔得？**
   答：可以但有風險：形成未確認鏈，父 tx 被替換或者卡住，子 tx 一齊無效／卡住；鏈太長亦超出 mempool 限制。一般限制只用已確認 UTXO，或者控制鏈長度。
3. **地址類型影響咩？**
   答：`1...` legacy、`3...` P2SH、`bc1q...` SegWit、`bc1p...` Taproot。SegWit/Taproot input 細啲、fee 平啲；提幣地址校驗要支援全部格式，充值地址一般派 bc1q。
4. **對帳點做？**
   答：鏈上以「熱錢包 UTXO 集合總和」對內部帳本；每張出金記錄用咗邊啲 input（鎖定唔俾第二單用），防止兩張出金搶同一粒 UTXO（雙花失敗）。
5. **同帳戶模型最大分別（對後端）？**
   答：冇 nonce，併發出金靠「UTXO 鎖定」而唔係 nonce 分配；餘額係算出嚟；fee 跟交易大小（input 數）而唔係 gas 用量。

## 小練習（附參考答案）
**題：** 一張 tx 有 3 個 output：vout0 → 用戶 A 充值地址 0.1 BTC，vout1 → 用戶 B 充值地址 0.05 BTC，vout2 → 外部找零。你點入帳？
**參考答案：** 監聽到後建兩條充值記錄：`txid:0` 俾 A 0.1、`txid:1` 俾 B 0.05，vout2 唔理。等夠確認數先入帳；reorg 時按 `txid:vout` 撤銷。

**題：** 出金 tx 開 2 sat/vB，卡咗 3 個鐘，用戶投訴。點處理？
**參考答案：** 如果有開 RBF：用同一批 input 重簽高 fee 版本（例如對齊當前 mempool 下一 block fee rate），出金單記錄新 txid，舊 txid 標 replaced。冇 RBF：用該 tx 嘅找零 output 做 CPFP。兩種都唔可以用另一批 UTXO 重新出金，否則有機會兩張都成功。
