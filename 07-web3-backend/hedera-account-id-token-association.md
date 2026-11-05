# Hedera：Account ID + Memo 入金、HTS Token Association

## 人話
Hedera 唔係「地址 = 公鑰 hash」咁簡單。帳戶係一個鏈上登記嘅編號 `0.0.12345`（shard.realm.num），要先「開戶」先存在。交易所通常用**一個收款帳戶 + memo** 分用戶（似 XRP destination tag / Stellar memo）。
代幣（HTS token）仲有一關：收款帳戶要先 **associate（關聯）** 個 token 先收得到，似 Stellar trustline / Algorand opt-in。

## 面試短答
- 入金：一個 exchange account + memo 識別用戶；掃鏈用 **Mirror Node REST API**（例如 `/api/v1/transactions?account.id=...`），唔係直接問共識節點。
- 冇 reorg：hashgraph 共識後幾秒就 final，唔使等確認數，但要等 Mirror Node 見到 `result=SUCCESS`。
- 冪等：交易 ID = `付款帳戶@validStart 時間戳`，有 validDuration（最長約 180 秒）窗口；同一 ID 重複提交會被拒，天然防重發。出金單要存住 transactionId。
- HTS token：收款方未 associate 就收唔到（`TOKEN_NOT_ASSOCIATED_TO_ACCOUNT`）；交易所收款帳戶上新幣前要先 associate。新版本有 auto-association（`maxAutoAssociations`，-1 = 無限），但唔好假設用戶帳戶有開。
- Token 可以有 **custom fee**（固定或按比例抽成）、KYC key、freeze key → 收到金額可能少過發送金額，要按實收入帳；上幣前要查 token 屬性。

## 常見追問（連答案）
1. **用戶出金去一個 EVM 風格 `0x...` 地址得唔得？**
   答：Hedera 支援 EVM alias，轉 HBAR 去未存在嘅 alias 會自動建一個「hollow account」。但出金前最好用 Mirror Node 查地址對應邊個 `0.0.x`、帳戶係咪存在；HTS token 去 alias 要睇對方有冇 association／auto-association 位，否則失敗但手續費照扣。
2. **出金失敗會唔會「卡住」？**
   答：唔會好似 EVM nonce 咁卡住後面嘅交易，因為冇順序 nonce；失敗會有明確 status（例如 `INVALID_ACCOUNT_ID`、`ACCOUNT_FROZEN_FOR_TOKEN`、`INVALID_SIGNATURE`）。過咗 validDuration 未上鏈就當過期，可以用新 transactionId 重發——但要先喺 Mirror Node 確認舊 ID 真係冇成功，避免雙發。
3. **`receiverSigRequired` 係咩？點影響出金？**
   答：有啲帳戶設定收款都要佢簽名，你單方面轉過去會失敗（`INVALID_SIGNATURE`）。出金遇到要標記為「目標帳戶唔接受」，退款畀用戶，唔好無限重試。
4. **memo 漏填點算？**
   答：同 XRP tag 一樣：入到總帳戶但認唔到人 → 進「待認領」隊列，用戶提交 tx ID + 證明人工認領；前端強制填 memo、memo 格式校驗。

## 小練習（題目 + 參考答案）
**題目**：你哋要上一個 HTS 穩定幣。測試網試入金，用戶話「已經轉咗」，但 Mirror Node 冇成功交易；另一筆成功嘅入金，用戶轉 100，你帳戶只收到 99.5。點解？點處理？

**參考答案**：
- 第一筆：好大機會交易所收款帳戶**未 associate 呢個 token**，交易失敗（`TOKEN_NOT_ASSOCIATED_TO_ACCOUNT`），錢根本冇出用戶帳戶（只扣咗手續費）。修正：上幣 checklist 加「收款帳戶 associate + 測試入金」，未 associate 前唔開充值。
- 第二筆：token 有 **custom fractional fee**（0.5% 抽成），或者有 fee collector。入帳一律按 Mirror Node 顯示**我方帳戶實際淨增加**（transfers / token_transfers 入面我方嗰行），唔好用用戶聲稱或者 tx 嘅「發送金額」。上幣前查 token info：custom_fees、kyc_key、freeze_key、pause_key，有呢啲 key 要評估風險（可以被凍結／暫停）。
