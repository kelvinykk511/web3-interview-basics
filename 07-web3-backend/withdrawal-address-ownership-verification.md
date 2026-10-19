# 出金地址所有權驗證（自託管錢包 / Travel Rule）

## 人話
用戶話「呢個地址係我自己嘅錢包」，交易所要佢**由嗰個地址出手**證明控制權：要麼用私鑰簽一段交易所出嘅 message，要麼由該地址轉一筆指定小額（Satoshi test）。交易所打錢過去**唔證明**任何控制權。

## 方法
- **簽名驗證（首選）**：
  - EVM EOA：`personal_sign`（EIP-191，加 `\x19Ethereum Signed Message:\n<len>` 前綴）或 EIP-712 typed data；後端 ecrecover 比對地址。
  - 合約錢包（Safe、ERC-4337）：冇私鑰可 ecrecover → `eth_call isValidSignature(hash, sig)`（EIP-1271），返回 `0x1626ba7e` 先算過；未部署嘅 counterfactual 錢包用 EIP-6492。要喺**對應條鏈**驗。
  - BTC：BIP-322（BIP-137 舊格式只支援 P2PKH，SegWit/Taproot 要 322）。
  - Solana：ed25519 `signMessage` 驗簽。
- **Satoshi test**：由該地址打指定奇怪金額（如 0.00012345）去交易所指定地址；成本高、慢，但唔使錢包支援簽 message。
- 對方係另一間 CEX（託管地址，簽唔到）：走 Travel Rule VASP 之間嘅訊息協議，唔係自託管驗證。

## Message 設計（防 replay / 釣魚）
- 必含：交易所 domain/名、uid、地址、chainId、伺服器生成一次性 nonce、簽發時間、過期時間（例如 10 分鐘）。
- nonce 用完即作廢；過期拒絕。
- 只用有前綴嘅 personal_sign / EIP-712，**唔好用 `eth_sign`**（簽任意 32 bytes，可以係交易 hash → 騙簽）。

## 驗證結果嘅有效期
- 唔係永久：EOA 可能被 EIP-7702 委託（getCode 出現 `0xef0100…`）、Safe owners 可改、合約錢包跨鏈 owner 可能唔同（counterfactual 地址唔假設跨鏈一致）。
- 加白名單 + 冷靜期（如 24h）；高風險/大額或者地址狀態改變時重新驗。
- 保存 message + 簽名 + 驗證結果做合規證據。

## 面試短答
「自託管地址要用戶由該地址出手證明控制權：EOA 用 personal_sign/EIP-712 + ecrecover，合約錢包用 EIP-1271 isValidSignature（未部署用 6492），BTC 用 BIP-322，唔支援就做 Satoshi test。message 要有 domain、uid、chainId、一次性 nonce 同過期時間防 replay，唔用 eth_sign 防騙簽。驗證係 per chain、會過期，加白名單冷靜期，證據留存畀 Travel Rule。」

## 常見追問（連答案）
1. **personal_sign 同 eth_sign 分別？** eth_sign 簽任意 hash，可能其實係一張交易嘅 hash → 用戶以為簽 message，實際授權咗交易；personal_sign 加固定前綴，簽出嚟唔會係有效交易簽名。
2. **合約錢包點驗？** 冇私鑰，ecrecover 冇意義；call 合約 `isValidSignature`，返回 magic value `0x1626ba7e` 先算；要喺同一條鏈 call。
3. **交易所打小額過去，用戶報返金額得唔得？** 唔得，只證明用戶見到條鏈（任何人都睇到），唔證明控制。要由該地址**發出**交易。
4. **驗咗一次係咪永久？** 唔係，7702 委託、owner 變更、key 外洩都會改變控制權；設有效期同觸發重驗。
5. **點解 message 要 chainId？** 同一 EVM 地址喺唔同鏈可以係唔同狀態（尤其合約錢包），簽名要綁死驗證嘅鏈。

## 小練習（附答案）
**題目**：設計 message 模板同後端（Java）驗證步驟。
**答案**：
```
ExampleEx 提幣地址驗證
Address: 0xAbC...
UserId: u_12345
ChainId: 1
Nonce: 8f3a2c9e...
IssuedAt: 2026-10-19T01:40:00Z
ExpiresAt: 2026-10-19T01:50:00Z
```
步驟：
1. 伺服器生成 nonce，存 DB（uid、地址、chainId、TTL 10 分鐘、unused）。
2. 收到簽名：檢查 nonce 存在、未用、未過期，message 內容同 DB 一致。
3. `eth_getCode(address)`：
   - 空 → EOA：web3j `Sign.signedPrefixedMessageToKey(msg.getBytes(UTF_8), sigData)` 還原公鑰，`Keys.getAddress(pubKey)` 同地址比（統一小寫；v 值 0/1 要轉 27/28）。
   - `0xef0100…` → 7702 委託 EOA：簽名仍可 ecrecover，但標記高風險、轉人工/加長冷靜期。
   - 其他 code → 合約錢包：`eth_call isValidSignature(hashMessage(msg), sig)` == `0x1626ba7e`。
4. 成功 → nonce 標記已用（同一事務），地址入白名單 + 24h 冷靜期，保存證據。
