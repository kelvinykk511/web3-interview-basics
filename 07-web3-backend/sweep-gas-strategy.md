# 歸集（Sweep）gas 策略

**人話**：每個用戶一個充值地址，錢散落幾萬個地址，要定期「掃」去熱/冷錢包。難點：ERC-20 地址冇 ETH 付 gas。

**面試短答**：按閾值+gas 價格觸發歸集；token 地址先由 gas 站打少量 ETH（gas 預估×緩衝），再 transfer；或用 CREATE2 forwarder/合約錢包批量 sweep 慳 gas。全程狀態機+冪等，唔影響用戶已入帳餘額。

**常見追問**
- 點解唔即時歸集？→ 小額 gas 可能大過金額；設最小金額閾值、低 gas 時段批量。
- 打 gas 打多咗？→ 殘留 ETH 係成本；按 estimateGas×maxFee 計，留少量 buffer，定期回收殘餘。
- 打 gas 後 token 未轉，gas 被用戶自己用掉？→ 地址私鑰喺交易所手，用戶冇得用；但要防重複打 gas（同地址有 PENDING 就唔再打）。
- 合約方案？→ CREATE2 預計算地址，有錢先部署 forwarder，或 factory 一個 tx 批量 flush 多個地址，免逐個打 gas。
- 帳本點處理？→ 歸集係內部資金搬運，不改用戶餘額；gas 記手續費成本科目。

**小練習**：USDT 充值地址有 3 USDT，gas 費約 2 USD，歸唔歸？
**參考答案**：唔歸。低於閾值（例如 gas 成本×5），等累積或 gas 平啲再批量；入帳早已按確認完成，歸集只關資金調度。
