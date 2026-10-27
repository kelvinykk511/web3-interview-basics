# BTC 地址類型 + 出金地址校驗（bech32 vs bech32m / Taproot）

## 人話
BTC 地址唔係一種，係幾種「鎖錢方式」嘅編碼：
| 前綴 | 類型 | 編碼 | scriptPubKey |
|---|---|---|---|
| `1...` | P2PKH（legacy） | Base58Check | `OP_DUP OP_HASH160 <20B> OP_EQUALVERIFY OP_CHECKSIG` |
| `3...` | P2SH（多簽/包住嘅 segwit） | Base58Check | `OP_HASH160 <20B> OP_EQUAL` |
| `bc1q...` | P2WPKH(20B)/P2WSH(32B)，witness v0 | **bech32**（BIP-173） | `OP_0 <20/32B>` |
| `bc1p...` | P2TR Taproot，witness v1 | **bech32m**（BIP-350） | `OP_1 <32B>` |

bech32 同 bech32m 只係 checksum 常數唔同（1 vs 0x2bc830a3）。規則：witness v0 必須 bech32，v1–v16 必須 bech32m。舊 library 只識 bech32 → 會話 `bc1p...` 無效。

## 面試短答
出金地址校驗要用 library 真正 decode，唔可以靠 regex：1) 按前綴判斷網絡（主網 `bc`/`1`/`3`，testnet `tb`/`m`/`n`/`2` 一律拒）；2) 驗 checksum（Base58Check 或 bech32/bech32m 按 witness version 揀）；3) 驗 program 長度（v0 只可 20 或 32 bytes，v1 Taproot 32 bytes）；4) 唔准大小寫混用；5) 未知 witness version（v2–v16）建議拒收，因為未有共識規則前屬 anyone-can-spend 風險。轉成 scriptPubKey 後先入出金單，DB 存 scriptPubKey 做 canonical key。

## 常見追問（連答案）
- **regex `^bc1[a-z0-9]{39,59}$` 夠唔夠？** 唔夠。冇驗 checksum，打錯一兩個字都過；bech32 保證檢出最多 4 個字元錯誤，regex 完全冇呢層保護。
- **唔同地址類型對出金成本有咩影響？** output 大小唔同：P2PKH ≈34 vB、P2WPKH ≈31 vB、P2SH ≈32 vB、P2WSH/P2TR ≈43 vB。批量出金估 fee 要逐個 output 計 vsize，唔好用固定值。
- **dust 門檻一樣嗎？** 唔一樣（Bitcoin Core 預設 dustRelayFee 3 sat/vB）：P2PKH 546 sats、P2WPKH 294、P2TR 330。最低出金額要按目標地址類型 ≥ dust，否則節點唔轉發。
- **充值地址用邊種？** 交易所自己揀，通常 P2WPKH（bc1q）平又兼容；就算充值唔支援 Taproot，**出金一定要支援送去所有標準類型**，否則用戶投訴「地址無效」。
- **掃描器點認 Taproot 充值？** 對 output 嘅 scriptPubKey `OP_1 <32B>` match 自家地址表；所以 DB 用 scriptPubKey（或 normalize 後全小寫地址）做 key，避免同一地址大小寫兩種寫法對唔上。

## 小練習（附參考答案）
**題：** 用戶提交 `bc1p...` 出金，系統報「地址格式錯誤」，客服叫用戶換地址。用戶話錢包係 Taproot。你點處理？
**參考答案：** 唔係用戶錯，係我哋 decoder 只支援 bech32（BIP-173），唔識 bech32m（BIP-350）。修正：升級 library，按 witness version 揀 checksum（v0→bech32，v1+→bech32m）；加 BIP-350 官方 test vectors 入單元測試（包括「v0 用 bech32m」同「v1 用 bech32」都要判無效）；fee 估算加 P2TR output 43 vB；最低出金額用 330 sats dust；未知 witness version 繼續拒收；修好後通知受影響用戶，並檢查有冇誤判其他合法地址。
