# ERC-4337／AA：CEX 充值點樣掃 UserOperation

> 銜接：基礎 4337 見 [`04-smart-contracts/erc4337-account-abstraction.md`](../04-smart-contracts/erc4337-account-abstraction.md)；同 7702 對比見 [`erc4337-vs-eip7702.md`](../04-smart-contracts/erc4337-vs-eip7702.md)。今日專注 **充值 indexer 漏單／錯歸因**。

## 人話（從你熟的 CEX）

你哋 indexer 慣咗兩件事：ERC-20 聽 `Transfer`；原生幣有時偷懶睇 `tx.to == 充值地址` 或者只掃 top-level value。  
用戶改用 **Smart Account（ERC-4337）** 之後，鏈上真實畫面係：

1. Bundler 發一筆普通 tx：`tx.from = Bundler`，`tx.to = EntryPoint`
2. EntryPoint `handleOps` 入面再 call 用戶帳戶 `execute`／`executeBatch`
3. ETH／token 先喺 **internal call** 入面落到你個充值地址

所以：**tx.from 唔係用戶、tx.to 唔係你**——但錢真係入咗。只靠 top-level 欄位就會漏單；反過嚟，若你聽 EntryPoint 事件就當入帳，又可能入錯帳。

## 面試短答

- **入帳真理來源不變**：ERC-20 → 我方地址嘅 `Transfer`（+ 確認數）；原生 ETH → **到我方地址嘅 value 轉移**（trace／callTracer／逐塊 balance-diff），唔係 UserOp 本身。
- **UserOperationEvent（EntryPoint）**：字段大致有 `userOpHash`、`sender`（Smart Account）、`paymaster`、`nonce`、`success`、`actualGasCost`。用途係 **歸因／對帳／客服**（邊個帳戶發起、邊個 Bundler 打包），**唔係** 入帳金額憑證。
- **Native ETH 漏單主因**：只索引 `tx.to`／receipt 頂層 `value`；4337／multisig／7702 batch 都會用 internal transfer。
- **Pending 唔入帳**：UserOp 喺 Bundler mempool 只係「意圖」；模擬通過 ≠ 上鏈成功；CEX 永遠等 finalized／達確認數嘅鏈上結果。
- **同一筆 handleOps**：一 tx 可包多個 UserOp；要按 log／internal call 拆分，唔好用「成筆 tx 成功」一次 credit 晒。

## 常見追問（連答案）

**Q1：點解 ERC-20 充值通常唔使改，原生 ETH 就要改？**  
A：ERC-20 標準靠 `Transfer` log，無論外層係 EOA、Bundler 定 EntryPoint，log 都會寫 `to=你地址`。原生 ETH 冇統一 Transfer event（除咗你自己合約 emit），頂層 receipt 又睇唔到 internal value，所以要 trace 或 balance-diff。

**Q2：可唔可以聽 `UserOperationEvent.success=true` 就入帳？**  
A：唔可以。成功只代表嗰個 UserOp 喺 EntryPoint 驗證＋執行流程冇 revert；執行內容可能係 approve、換倉、轉去別人，唔一定轉畀你。金額、資產、收款地址都要以實際 token／ETH 流動為準；事件只幫忙把 `userOpHash`／`sender` 掛去用戶檔案。

**Q3：CEX 要支援「AA 用戶充值後喺 App 顯示來源錢包」，點設計？**  
A：入帳流水主鍵仍然係 `(chainId, txHash, logIndex|traceId)`。額外欄位：若同塊／同 tx 內搵到 EntryPoint `UserOperationEvent`，把 `sender` 寫做 `onchain_source_account`；`tx.from` 標成 `bundler`。UI 顯示 Smart Account 地址，唔好顯示 Bundler。對唔上事件就顯示「合約內部轉入」，唔好瞎猜。

**Q4：Paymaster 贊助嘅充值，gas 費會唔會搞亂你對帳？**  
A：Gas 由 EntryPoint 從帳戶 deposit 或 Paymaster 扣，同用戶轉畀你嘅本金係分開嘅。你帳本只 credit 收到嘅資產數量；`actualGasCost` 最多做成本監控，唔好從入帳金額扣走（除非產品明確做「gas 代扣」而且鏈上真係咁轉）。

## 小練習（題目 + 參考答案）

**題目**：現有 ETH 充值 scanner 用「`tx.to == depositAddress && value > 0`」；上線後 AA 用戶投訴到帳慢／漏單。你會點改架構？另一個同事話「改聽 EntryPoint 全部事件入帳就得」。你點回應？

**參考答案**：
- **立刻改 indexer**：對每個監控地址，按塊做 (1) ERC-20 `Transfer` 到地址；(2) native：`debug_traceBlockByHash`／callTracer（或节点提供的 internal tx）篩 `to=deposit && value>0`，失敗節點就 fallback **塊末 balanceOf − 已知出金** 做 anomaly 再人工／重掃。冪等鍵用 trace 唯一路徑或 `(txHash, traceAddress)`。
- **拒絕「只聽 EntryPoint 入帳」**：事件無資產／收款語意，而且會把非充值 UserOp 也掃入隊列。正確係 **錢流先行、UserOp 事件側掛**。
- **狀態機**：`seen → confirming → credited` 不變；pending UserOp／mempool 通知最多做「預計中」灰字，**永不 ledger credit**。
- **回歸用例**：Bundler `handleOps` 一次轉 ETH 入專屬充值地址；batch 兩個 UserOp 只其一轉畀你；失敗 UserOp（`success=false`）唔入帳；普通 EOA 直轉仍要通過。
