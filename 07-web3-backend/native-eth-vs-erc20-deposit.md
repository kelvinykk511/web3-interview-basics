# Native ETH vs ERC-20 充值偵測（CEX）

## 人話（從你熟的 CEX）

用戶充 USDT／大部分 token：鏈上會噴出 `Transfer` log，你哋 indexer 用 `eth_getLogs`／subscribe 撈事件，解碼 `to`／`value` 就夠入帳候選。

用戶充 **原生 ETH**（或 BNB、MATIC 等鏈原生幣）：普通轉帳 **冇 ERC-20 嗰條 Transfer event**。錢喺 transaction 嘅 `value` 欄（或者合約內部 `CALL` 轉 value）。若你只掃 ERC-20 logs，ETH 充值會「隱形」——帳本永遠加唔到，客服卻話瀏覽器見到入咗。

類比：有啲支付渠道有標準 webhook JSON；有啲只係銀行電匯、要自己對流水／餘額——同一「充值」產品，兩套觀測路徑。

## 面試短答

- **ERC-20**：主路徑 = 白名單合約嘅 `Transfer` logs；金額用事件 `value`（FoT 再對 `balanceOf` 增量）。
- **Native**：主路徑 = 掃 block／tx：`to ∈ 充值地址集` 且 `value > 0`；receipt `status=1`；再加確認數。
- **合約錢包／路由入金**：用戶可能經合約轉 ETH，外層 tx 嘅 `to` 唔係你充值地址，真正入帳靠 **internal call**（要 trace／專用 internal-tx 索引，或對地址做 balance 增量對帳）。
- **冪等鍵**：ERC-20 常用 `txHash + logIndex`；native 直轉常用 `txHash`（一筆 tx 一個 value 到 EOA）；若要統一模型可虛擬成 `logIndex=-1` 或 `kind=native`。
- **產品**：上幣表要標 `assetType=native|erc20`，掃鏈管道按類型分支，唔好一套 filter 打天下。

## 常見追問（連答案）

**Q：點解唔可以對充值地址只 poll `eth_getBalance`？**  
A：能發現「餘額變咗」，但唔知係邊筆、邊個用戶（HD 多地址仲好辦；共用熱錢包地址就難）、幾多確認、同有冇同時 ERC-20 入帳。Balance 適合對帳／告警，唔適合單獨做入帳真相；入帳要綁具體 tx（同可選 internal trace）。

**Q：`eth_getTransactionByHash` 見到 `value>0` 且 `to=我哋地址`，夠未？**  
A：候選可以。仍要：正確 chainId、receipt success、確認數、金額 ≥ 最小入帳、地址屬於該用戶／充值表、冪等防重放。合約創建交易（`to=null`）唔係充值。

**Q：用戶用合約（例如交易所／bridge 聚合器）轉 ETH 入我哋地址，外層 tx.to 唔係我哋，點辦？**  
A：plain tx 掃描會漏。選項：(1) 支援 internal tx 索引（`debug_traceBlock`／第三方 internal API，成本高）；(2) 產品引導「請用錢包直接轉，唔好經某某路由」；(3) 對單地址做 balance-diff + 人工／工單補入（差體驗）。CEX 常見係 (2)+(1 有限範圍熱路徑)。

**Q：同一 tx 同時轉 ETH value 又 emit ERC-20 Transfer，會唔會雙重入帳？**  
A：兩條管道要分資產類型入帳；ETH ledger 同 USDT ledger 分開。唔好用「一個 txHash 只許入一次」跨幣種誤傷；冪等鍵要帶 `asset`／`contract`（native 用 `address(0)` 慣例）。

## 小練習

**題：** 掃鏈只訂閱咗 USDT／USDC 嘅 `Transfer`。用戶向 HD 充值地址打咗 0.5 ETH（普通 EOA→EOA，`value=0.5e18`，成功上鏈 20 確認）。系統現象係咩？要點改架構先入到帳？若同一用戶其實係經一個「轉發合約」打入（外層 `to=Forwarder`），仲有咩額外風險？

**參考答案：**  
1. 現象：ERC-20 indexer 完全無事件 → 無自動入帳；用戶瀏覽器見到 ETH 到帳，客服工單爆。  
2. 改造：為 native 資產加 tx／block 掃描（或 `newPendingTransactions` 僅作預警，入帳仍靠 mined+確認）；配置 `assetType=native`；冪等 `chainId+txHash+asset`；確認數策略可同 ERC-20 分開配。  
3. 經 Forwarder：外層 tx.to ≠ 充值地址 → plain 掃描仍漏；要嘛拒支持（產品文案），要嘛上 internal-tx／trace 管道並嚴格限制可識別 pattern，避免把任意 internal 當充值導致誤入帳。
