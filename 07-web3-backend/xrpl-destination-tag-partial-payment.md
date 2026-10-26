# XRP Ledger 充值：Destination Tag + Partial Payment 攻擊

## 人話
XRP 唔係每個用戶一個地址，通常交易所共用一個收款地址，用 **Destination Tag**（32 位無符號整數）分辨邊個用戶，類似其他鏈嘅 memo。最大坑係 **Partial Payment**：交易入面嘅 `Amount` 只係「最多交付」，如果帶咗 `tfPartialPayment` flag，實際到帳可以少好多。真正到帳要睇 metadata 嘅 `delivered_amount`。

## 面試短答
XRPL 充值：只處理 `validated: true` 嘅 ledger（XRPL 共識幾秒就最終，冇 reorg），而且 `TransactionResult` 必須係 `tesSUCCESS`；`tec*` 結果都會上鏈收手續費，但冇轉帳，唔可以入帳。入帳金額用 `meta.delivered_amount`，唔用 `Amount`，防 partial payment 攻擊。用 Destination Tag 對應用戶，收款地址開 `RequireDest` 拒收冇 tag 嘅交易。代幣（IOU，例如某 issuer 發嘅 USD）要核對 issuer 地址同 currency code，防假幣。冪等鍵用 tx hash。

## 重點清單
- **delivered_amount**：Payment 加 `tfPartialPayment` 時，`Amount` 可以寫 10,000 XRP，實際只交付 0.01。只認 `meta.delivered_amount`（`unavailable` 嘅舊交易就唔自動入帳）。
- **tec vs tes**：`tes` = 成功；`tec` = 失敗但已上鏈收咗 fee；`tef/tem/ter` 未上鏈。充值只認 `tesSUCCESS`。
- **RequireDest**：帳戶 flag `asfRequireDest`，冇 tag 嘅 payment 會被拒，減少無主充值。
- **Reserve**：帳戶要有 base reserve 先激活（2024 年 12 月起 base 1 XRP、每個 object 0.2 XRP），出金去未激活地址少過 reserve 會 `tecNO_DST_INSUF_XRP`。
- **IOU / trust line**：XRPL 上任何人都可以發「USD」，要用 (currency, issuer) 做幣種 key。
- **出金**：`Sequence` 類似 nonce；設 `LastLedgerSequence`，過咗呢個 ledger 仲未入 = 確定失敗，可以安全重發，唔會雙花。

## 常見追問（連答案）
**Q1：充值收到 tec 結果點處理？**
A：唔入帳。佢上鏈只係扣咗發送方 fee，冇資金轉移。出金側收到 tec 要當失敗按原因處理（例如目標地址未激活就退款給用戶），下一筆用新 Sequence。

**Q2：用戶冇填 tag 點算？**
A：預防：收款地址開 RequireDest。如果仍然收到（例如舊地址），入無主充值隊列，用戶提交 tx hash + 身份證明人工認領。

**Q3：LastLedgerSequence 有咩用？**
A：俾每筆出金一個「確定性失敗」邊界。驗證過的 ledger 已經超過 LastLedgerSequence 而交易未入，就肯定永遠唔會入，可以安全重簽重發，避免同一張出金單發兩次。

**Q4：點解 XRP 可以「幾秒最終」唔使等確認數？**
A：XRPL 用共識協議，ledger 一被 validated 就最終，冇 PoW 嘅分叉重組。但仍然要以 validated ledger 為準，唔好用 current/closed 未驗證數據。

## 小練習（附參考答案）
**題目**：攻擊者發一筆 Payment 到你嘅充值地址，`Amount` = 10,000 XRP，`Flags` 包含 `tfPartialPayment`，實際 `delivered_amount` = 0.01 XRP，`TransactionResult` = tesSUCCESS。你個系統點樣先唔會中招？之後佢再申請提現 10,000 XRP 又點？

**參考答案**：
1. 入帳邏輯只讀 `meta.delivered_amount`，所以只入 0.01 XRP；`Amount` 欄位完全唔用嚟入帳。
2. 額外防線：見到 `tfPartialPayment` 標記嘅充值打 tag 告警，大額差異（Amount 遠大於 delivered）入風控。
3. 對帳：每日比對收款地址鏈上 XRP 餘額同帳本負債，任何多入帳會即時暴露。
4. 佢申請提現 10,000 時，帳本可用餘額只有 0.01，直接拒絕；即使之前誤入帳，出金前嘅餘額對帳/風控大額審批都應該攔截。
5. 出金自己發嘅 Payment 唔好開 partial payment flag，避免用戶少收。
