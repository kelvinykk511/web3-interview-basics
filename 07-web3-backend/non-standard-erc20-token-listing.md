# 上新幣技術審核：非標準 ERC-20（fee-on-transfer / rebasing / 唔返 bool / 可升級）

## See also

- [非標準 ERC-20：fee-on-transfer／USDT／blacklist](../04-smart-contracts/erc20-nonstandard-quirks.md)
- [新幣上架：鏈上盡調 Checklist](./token-listing-onchain-checklist.md)

## 人話
ERC-20 只係「約定」，唔係「保證」。好多 token 唔守規矩：轉帳會抽水、餘額自己會變、`transfer` 唔返 `true`、合約可以被 admin 換晒邏輯。CEX 如果假設「Transfer event 嘅 amount = 用戶實收 = 我錢包多咗嘅錢」，遲早帳對唔上。

## 面試短答
上幣前做技術審核：(1) 讀合約／喺 fork 鏈實測 transfer，對比 Transfer 事件金額同 `balanceOf` 差額，偵測 fee-on-transfer；(2) rebasing token（AMPL、stETH）餘額會自己變、冇 Transfer 事件，要嘛唔支持，要嘛只收 wrapped 版（wstETH），要嘛按 shares 記帳；(3) 唔返 bool 嘅 token（Ethereum 上嘅 USDT）用 SafeERC20 式處理，以 receipt status + 餘額為準；(4) proxy 可升級／可 pause／有黑名單嘅 token，監聽 `Upgraded`、`AdminChanged`、`Paused` 事件，一變就自動暫停該幣充提、人工 review；(5) decimals 按 (chain, contract) 配置，用 BigInteger 計，唔用 double。線上再用「事件流水 vs 鏈上餘額」對帳兜底。

## 重點清單
- **Fee-on-transfer**：有啲 token 轉 100 收款方只收 98。事件可能報毛額或淨額，唔好估，實測。入帳以實際到帳為準；出金時用戶會少收，要喺 UI 提示，熱錢包扣款要計埋 fee。
- **USDT 隱藏 fee 參數**：Ethereum USDT 合約有 `basisPointsRate`/`maximumFee`，現時係 0，但 owner 可以開 → 要監控參數變化。
- **Rebasing**：AMPL 餘額每日按比例變；stETH 每日加獎勵，內部按 shares 記，`transfer` 有 1–2 wei 四捨五入少收 → 歸集時差 1–2 wei 唔好當異常。
- **唔返 bool**：舊 USDT `transfer` 冇返回值，用標準 interface `require(token.transfer(...))` 會 revert；Java 側用 web3j 解碼時唔好假設有 output。
- **可升級 proxy**：今日正常，聽日升級可以加 fee、加黑名單。上幣要記錄 implementation 地址，變咗就告警。
- **decimals**：USDC 6、多數 18；同一個幣喺唔同鏈 decimals 可以唔同（例如 BSC 上嘅 USDT 係 18）。
- **ERC-777 hooks**：收款會觸發回調，主要係合約重入風險，對 CEX 帳本影響較細，但要知。

## 常見追問（連答案）
**Q1：點樣自動偵測 fee-on-transfer？**
A：上幣時用 fork 鏈（如 anvil --fork-url）由有餘額地址轉一筆去測試地址，比較 `balanceOf` 前後差額同 Transfer 事件金額；線上入帳後抽樣對比「事件累計」同「地址實際餘額」，唔一致就告警並暫停該幣。

**Q2：用戶充 1 stETH，歸集時發現地址只有 0.999999999999999999？**
A：stETH 按 shares 計，轉帳有 1–2 wei 捨入誤差，屬正常。帳本要有 dust 容忍度；更乾淨做法係只支持 wstETH（非 rebasing，餘額唔會自己變）。

**Q3：token 合約被升級點處理？**
A：監聽 proxy 嘅 `Upgraded(implementation)` / `AdminChanged` 事件，觸發即自動暫停該幣充提，人工審核新 implementation（有冇加 fee、黑名單、mint 權限）先恢復。

**Q4：點解唔可以只信 Transfer event？**
A：rebasing 冇事件、黑名單 destroy 冇事件、fee 事件可能報毛額、惡意合約可以亂 emit 事件。事件係「線索」，餘額係「事實」，兩樣都要對。

## 小練習（附參考答案）
**題目**：用戶充 100 個某新 token，Transfer 事件顯示 100，你入咗帳 100。歸集時發現充值地址只有 98。點排查、點處理？

**參考答案**：
1. 先停該幣充值入帳同出金，防止損失擴大。
2. 排查：查該 tx receipt 全部 logs（係咪有第二條 Transfer 去 fee 地址）、用 `eth_call balanceOf` 喺該 block 前後查餘額、讀合約源碼（fee 邏輯、rebase、黑名單、最近有冇 `Upgraded`）。
3. 如果係 fee-on-transfer：改入帳邏輯為按實際到帳（餘額差額或淨額事件）入帳；已多入帳嘅部分按公告規則處理（扣回或交易所承擔，要合規/客服流程）。
4. 如果係合約升級後新加 fee：補上 implementation 監控。
5. 長期：上幣 checklist 加 fork 實測，每日鏈上 vs 帳本對帳覆蓋所有幣。
