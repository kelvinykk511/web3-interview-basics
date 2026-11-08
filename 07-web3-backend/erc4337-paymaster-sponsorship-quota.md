# ERC-4337 Paymaster：贊助配額同 CEX onboarding

> 銜接：4337 總覽見 [`04-smart-contracts/erc4337-account-abstraction.md`](../04-smart-contracts/erc4337-account-abstraction.md)；尋日掃 UserOp／counterfactual 派址見 [`erc4337-cex-deposit-userop-scan.md`](./erc4337-cex-deposit-userop-scan.md)、[`erc4337-counterfactual-deposit-address.md`](./erc4337-counterfactual-deposit-address.md)。今日專注 **邊個代付 gas、點樣唔俾人刷爆預算**。

## 人話（從你熟的 CEX）

普通 tx 係 `tx.from` 自己出 ETH 付 gas。AA 帳戶可以一個 ETH 都冇——首次 deploy、綁定、寫 session key 仍然要上鏈。  
**Paymaster** 就好似「新客手續費優惠券」：UserOperation 帶住 `paymasterAndData`，EntryPoint 驗證通過之後，改由 Paymaster 喺 EntryPoint 入面嘅 **deposit** 扣 gas，用戶唔使事先持有原生幣。

交易所若要做「用 Smart Account 註冊／首次上鏈都唔使有 ETH」，通常會自己跑 **Verifying Paymaster**（鏈下審批、鏈上只驗簽名），而唔係無條件幫全世界付 gas。呢張券一旦冇額度、冇到期、冇允許做嘅動作清單，就等如公開資助攻擊者燒你庫房。

## 面試短答

- **流程**：Bundler 包 UserOp → EntryPoint 先 `validateUserOp`，若有 Paymaster 就 call `validatePaymasterUserOp(userOp, userOpHash, maxCost)` → 執行 `callData` → 再 `postOp`。Gas 從 Paymaster（或帳戶自己）喺 EntryPoint 嘅存款扣，**同用戶轉俾你嘅本金分開**。
- **兩類常見 Paymaster**：
  - **Verifying**：後端對呢一個 UserOp 簽批（deadline、nonce、gas 上限），鏈上 `ecrecover`。CEX onboarding 多數用呢種，因為配額、KYC、allowlist 都喺鏈下做。
  - **ERC-20／deposit**：用戶用 USDC 等代幣付 gas，Paymaster 收代幣、墊 ETH。呢個係「用穩定幣付手續費」產品，唔係純補貼。
- **`paymasterAndData`**：前 20 bytes 係 Paymaster 地址，後面係 payload（簽名、`validUntil`／`validAfter`、配額 id 等）。
- **EntryPoint deposit ≠ stake**：真正扣 gas 靠 **deposit**（要監控水位、及時充值，好似熱錢包 gas 水位）。**stake** 主要係 Bundler 抗 DoS／信譽：未 stake 或者聲譽差嘅 Paymaster，好多 Bundler 會拒收。
- **配額要按 userId 而唔係只按 sender**：一人可以 CREATE2 出無限個 sender。還要有 **單筆 maxCost**、**每日／每月上限**、**全局日預算**、以及 **callData allowlist**（通常只准 factory 部署帳戶、綁定、寫指定 session，唔准任意 `execute`／swap／無限 `approve`）。
- **失敗都會燒錢**：驗證通過但執行 revert，Paymaster 好多時仍要付驗證＋部分執行 gas（`postOp` 的 `opReverted`）。預算唔可以假設「失敗就免費」。
- **簽名要短命同防重放**：`validUntil` 以分鐘計；綁 `userOpHash` 或一次性 nonce，避免同一張券被改 gas／重放。簽出券嘅一刻先預扣（reserve）配額，TTL 內未上鏈再釋放。
- **Paymaster 私鑰被盜 = 可抽乾 EntryPoint deposit**。緩解：MPC／HSM 保管、即使鑰被盜都只簽得中 allowlist 嘅平價操作、鏈下日額熔斷、deposit 低水位告警。

## 常見追問（連答案）

**Q1：CEX 贊助 onboarding，最少應該允許邊幾種 UserOp？**  
A：白名單思維：`initCode` 只接受自己支援嘅 factory＋salt／owner 參數；`callData` 只接受「部署後綁定／寫一條指向本所嘅 session／登記模組」呢類 selector。唔好贊助任意目標、任意 selector。首次充值本身通常唔使你代付——用戶把幣轉入 counterfactual 地址即可（尋日講過，未部署都收得）。

**Q2：點解唔可以「新地址 24 小時內任意 UserOp 都贊助」來換順暢 UX？**  
A：攻擊者可以大量建立新 sender，用高 `callGasLimit`、寫 storage、或者專門令執行失敗但仍耗驗證 gas，把你 deposit 燒光。無 KYC 綁定更加無法事後追討。UX 應該靠「少而準嘅允許動作」，唔係打開任意執行。

**Q3：配額應該喺簽名時扣，定等 `UserOperationEvent.success=true` 先扣？**  
A：簽批時 **reserve**，過期未上鏈再釋放；成功或失敗上鏈都按實際消耗結算（失敗都可能扣 gas）。如果只喺 success 先扣，攻擊者可以反覆要券、模擬、丟棄，令你簽簽器同 Bundler 被刷。事件只用來對帳 `actualGasCost`，唔好當「有事件先算贊助成功」嘅唯一記帳點。

**Q4：Paymaster 贊助會唔會令充值金額少咗一截？**  
A：唔會，除非你產品另外做「從入帳扣手續費」而且鏈上真係咁轉。EntryPoint 扣嘅係 Paymaster deposit 入面嘅 ETH gas，同 `Transfer`／native value 到你充值地址嘅本金係兩條流水。客服對帳：本金睇 token／ETH 流動；gas 成本睇 `actualGasCost`＋自己嘅贊助帳本。

**Q5：Bundler 拒收我哋 Paymaster 通常係咩原因？**  
A：未喺 EntryPoint stake、deposit 不足、模擬時 `validatePaymasterUserOp` 唔穩定（同一 UserOp 有時過有時唔過）、或者聲譽／白名單未納入該 Bundler。生產要：足夠 stake＋deposit、驗證邏輯純函數（唔好依賴會變嘅外部價格而無超時保護）、同至少一個備援 Bundler。

## 小練習（題目 + 參考答案）

**題目**：產品話「AA 新客第一次上鏈（deploy 帳戶＋綁定 CEX）唔使自己有 ETH」。同事提議做一個 Paymaster：任何未見過嘅 `sender`，24 小時內所有 UserOp 都代付，而且唔檢查 `callData`，「這樣註冊轉化率最高」。你會點設計？邊幾條防線必加？

**參考答案**：
- **拒絕無限制贊助**。改用 Verifying Paymaster：只有已 KYC 的 `userId` 先簽批；每張券綁 `userOpHash`（或一次性 nonce）＋短 `validUntil`（例如 2–5 分鐘）＋ `maxCost` 上限。
- **Allowlist**：`initCode` 只准指定 factory；`callData` 只准 deploy／bind／寫本所 session 嘅 selector 同目標合約。任意 `execute`、swap、`approve(spender, max)` 一律拒簽。
- **額度**：按 `userId` 每日次數（例如 3 次 onboarding 類 UserOp）＋單筆 maxCost＋全局日預算；簽出時 reserve，過期釋放；監控 EntryPoint deposit，低水位告警同自動停簽（熔斷）。
- **成本現實**：執行 revert 都可能扣 gas，預算按「最壞 maxCost」計，唔按「成功先計」。
- **鑰匙**：簽批鍵用 HSM／MPC；即使被盜，allowlist 令攻擊者都簽唔到高成本任意操作。回歸用例：正常 deploy+bind 成功；改 `callData` 去 swap 被拒；同一張券重放被拒；deposit 低於閾值時新請求直接失敗並告警。
