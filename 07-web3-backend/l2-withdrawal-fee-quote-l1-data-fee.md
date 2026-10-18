# L2 出金手續費報價：L1 data fee 點樣影響成本（OP Stack vs Arbitrum）

## 人話
喺 L2 出一筆錢，CEX 實際俾嘅 gas 有兩份：L2 執行費（好平）+ 將 tx 數據寫返 L1 嘅成本（L1 data fee，通常係大頭）。如果出金費只按 L2 gas 計，L1 一貴就蝕。

## 兩大家族點計
- **OP Stack（Optimism / Base 等）**：L1 data fee 係獨立一份。receipt 有 `l1Fee`、`l1GasUsed`、`l1GasPrice`、`l1BlobBaseFee`、scalar 等欄位。`eth_estimateGas` **唔包** L1 fee。預估用 predeploy `GasPriceOracle`（`0x420000000000000000000000000000000000000F`）嘅 `getL1Fee(bytes)`。Ecotone 之後用 L1 base fee + blob base fee × scalar；Fjord 之後按壓縮後大小（FastLZ）估。
  - 實際成本 = `gasUsed × effectiveGasPrice + l1Fee`。
- **Arbitrum**：L1 成本摺入 `gasUsed`（receipt 有 `gasUsedForL1`），所以 `eth_estimateGas` 已包；但 L1 價一升，同一筆 transfer 嘅 gasUsed 都會變大 → gas limit 唔可以寫死。L2 gas price 平時貼近 floor。
  - 實際成本 = `gasUsed × effectiveGasPrice`（已含 L1 部分）。

## CEX 報價策略
1. 定時（例如每 30–60 秒）用一筆樣本出金 tx（同類 token、同 calldata 長度）估總成本（原生幣計）。
2. 換算成提幣幣種（例如 USDT）：價格源要防 stale。
3. × buffer（例如 1.2–1.5）+ 平滑（EMA）+ 設上下限，避免報價跳嚟跳去。
4. 用戶確認時鎖價一段 TTL；實際成本 > 報價時照出（CEX 吸收）但記錄＋告警。
5. L1 尖峰（blob base fee 指數調價）超過上限 → 暫停該鏈出金或提示延遲。
6. 熱錢包要有原生幣 gas 補倉（gas tank）監控。

## 帳本 / 對帳
- 記帳用 **receipt 實際值**，唔用估算值。
- OP Stack 對帳要加埋 `l1Fee`，否則熱錢包原生幣餘額永遠對唔上。
- 長期監控「收費 ÷ 實際成本」比率，按鏈調 buffer。

## 面試短答
L2 出金成本 = L2 執行費 + L1 data fee。OP Stack 嘅 l1Fee 獨立計，estimateGas 唔包，要 call GasPriceOracle.getL1Fee 或睇 receipt 嘅 l1Fee；Arbitrum 將 L1 成本摺入 gasUsed，所以 gas limit 唔可以寫死。我哋定時用樣本 tx 估總成本、換算幣種、加 buffer 同平滑，報價鎖 TTL，記帳用 receipt 實際值，再監控收費同成本比率。

## 常見追問（連答案）
- **Q：OP 鏈上熱錢包扣多過 gasUsed × gasPrice，係咪漏數？** A：唔係，多咗 l1Fee，喺 receipt 有，對帳要加埋。
- **Q：Arbitrum 同一筆 USDT transfer 今日 gasUsed 高過尋日好多，係咪 bug？** A：唔係，L1 部分摺入 gasUsed，跟 L1 價浮動；gas limit 用 estimateGas × buffer。
- **Q：出金費收提幣幣種定原生幣？** A：通常按提幣幣種收（例如 USDT），CEX 用原生幣付 gas，要匯率換算 + buffer，價格源防 stale。
- **Q：L1 突然貴 10 倍點做？** A：報價 TTL 到期自動更新；超上限暫停或延遲該鏈出金；已確認嘅單按鎖價出，差額 CEX 吸收並告警。

## 小練習（附答案）
OP Stack 鏈出一筆 USDT：gasUsed 65,000，effectiveGasPrice 0.002 gwei，receipt `l1Fee` = 0.0000012 ETH。總成本幾多？

**答案**：L2 部分 = 65,000 × 0.002 gwei = 130 gwei = 0.00000013 ETH；總成本 = 0.00000013 + 0.0000012 = 0.00000133 ETH，L1 部分佔約 90%。（數字係練習用嘅示例數據。）
