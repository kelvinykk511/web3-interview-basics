# Multisig + Timelock（CEX 金庫／合約 Admin）

## 人話（從你熟的權限）

CEX 後端有「雙人覆核出款」「變更提現白名單要審批」——鏈上對應就係 **Multisig**（例如 Safe：M-of-N owners）同 **Timelock**（提案通過後要等一段 delay 先執行）。  
昨日講 proxy 升級／owner 係單 EOA 好危險；今日補「生產應該點擺權限」。熱錢包簽名可以係 HSM／MPC；**治理／升級／大額歸集目標變更** 通常另走 multisig + timelock。

## 面試短答

- **Multisig**：一筆操作要湊夠 M 個簽名先執行；降低單點私鑰失竊即時盜光。
- **Timelock**：`queue` → 等 `minDelay` → `execute`；畀監控／用戶／風控有時間發現惡意提案並暫停充提。
- 典型分層：熱錢包（限額自動出）／冷或多簽大額／合約 `owner`／`DEFAULT_ADMIN_ROLE` 放 Safe；`upgradeTo`／`setBridge` 再包 Timelock。
- 事件：盯 `AddedOwner`、`RemovedOwner`、`CallScheduled`、`CallExecuted`、`MinDelayChange`——同你盯 DB 權限變更一個道理。

## 常見追問（連答案）

**Q：Multisig 係咪就等於安全？**  
A：唔係。N 個 owner 私鑰一齊放同一雲、或社工一次過騙齊簽，一樣爆。要：地理／人員分離、硬體／MPC、閾值合理（例如 3/5 唔好 1/1）、同 recovery 流程演練。

**Q：Timelock delay 點揀？**  
A：睇「發現惡意變更 → 暫停充提／遷移」要幾耐。常見 24h–72h；緊急 pause 可另設 **Guardian／Pauser** 多簽（可即時暫停、唔畀即時升級）。面試金句：pause 要快、upgrade 要慢。

**Q：同「後端出款 dual-control」點類比？**  
A：dual-control ≈ 多簽閾值；变更窗口 ≈ timelock；audit log ≈ 鏈上事件 + 內部工單。分別：鏈上提案人人可見，監控可以公開化；但 delay 期間資金／邏輯仍可能被舊漏洞打——timelock 唔替代審計同限額。

## 小練習

**題：** 自研歸集合約而家 `owner` 係部署用 EOA。要上生產，你會點遷移權限？遷移期間充提點辦？

**參考答案：**  
(1) 部署／準備 Safe（3/5）同 Timelock；(2) EOA 只做一次 `transferOwnership(timelock)` 或把 admin role 授畀 Timelock，Timelock 嘅 admin／proposer 設為 Safe；(3) 驗證 on-chain：owner／role 已唔係 EOA，並用測試 `queue` 一筆 harmless 操作走完 delay；(4) 遷移窗口：暫停升級類函數或整體 pause、限制熱錢包敞口、監控 OwnershipTransferred。確認後先恢復正常充提；舊 EOA 金鑰作廢並記錄。
