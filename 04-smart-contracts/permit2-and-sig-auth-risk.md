# Permit2 與簽名授權提現風控

## 人話（從你熟的 CEX）

EIP-2612 係 **每個 token 自己** 實作 `permit`。Uniswap **Permit2** 係一條共用合約：用戶先對 token `approve(Permit2)`（一次），之後用 **鏈下簽名** 畀某個 spender 在額度／期限內 `transferFrom`——好多 DeFi／跨協議授權都走呢條。

對 CEX 後端面試：重點唔係背 Permit2 介面細節，而係 **簽名授權＝把「可扣款權」變成可傳遞嘅憑證**。用戶（或內部運維）簽錯 spender／deadline／額度，或者簽名洩漏，後果似 API key 洩漏加提幣權限。要同「熱錢包直接 transfer 出金」「2612 permit」「假授權釣魚」分開講。

## 面試短答

- **Permit2 模型**：token → 批准 Permit2 合約為 spender；其後 `permit`／`permitWitnessTransferFrom` 等用 EIP-712 簽名描述 `spender、amount、expiration、nonce`；任何人可代交上鏈，合約核簽後允許 spender 拉幣。
- **vs EIP-2612**：2612 要 token 實作；Permit2 **一次 approve 多用**、跨 token 統一簽名格式；亦因此攻擊面集中：用戶對 Permit2 嘅 allowance 同簽名域要當高敏感。
- **CEX 相關場景**：(1) 用戶從交易所提幣去 DeFi 後被騙簽 Permit2；(2) 產品若做「簽名代扣／一鍵授權」，後端必須校 domain（name、chainId、verifyingContract＝Permit2 地址）、spender 白名單、amount、deadline；(3) 內部若用簽名授權做出金／調撥，等同第二套提現金鑰——要過風控、短 deadline、單次 nonce、可撤銷。
- **風控要點**：展示明文（spender 合約是誰、額度是否 max、過期時間）；禁無限＋超長 deadline 嘅預設；簽名後監控 `Approval`／Permit2 事件；支援 revoke；提現地址／授權分開產品路徑，唔好混在「只係簽名登錄」。

## 常見追問（連答案）

**Q：用戶已經 approve 咗 Permit2，係咪等於永遠俾人轉走幣？**  
A：唔係自動。Permit2 仲要有效簽名（或既有 allowance 條目）先畀具體 spender 拉幣。但 approve(Permit2)=max 令「之後任何成功騙簽」都可以在限額內轉走——所以要教育／產品預設有限額，並提供一鍵 `approve(0)`／Permit2 revoke。

**Q：點解面試常問「簽名授權」同「鏈上提現」一齊？**  
A：兩者都可以導致資產離開用戶或平台控制。鏈上提現睇地址＋確認數；簽名授權睇 **typed data 內容**。釣魚站常偽裝成「驗證錢包／領空投」，實際叫你簽 Permit2／2612／`eth_sign` 任意訊息。CEX 客服／風控要識辨：用戶說「我冇提幣但幣冇咗」時查授權事件，唔只查 Withdraw 記錄。

**Q：若 CEX 自家做「用戶簽一次，平台代扣手續費／兌換」？**  
A：必須：固定 `verifyingContract`、短 `deadline`、amount 精準、spender＝自有合約、nonce 單次；後端驗證簽名後先上鏈，狀態機冪等；禁止前端把「登錄簽名」複用做出金授權（訊息域分離）。私鑰／簽名服務同熱錢包出金一樣要 HSM、審計、額度熔斷。

**Q：Permit2 簽名同 EIP-55／地址投毒有冇疊加？**  
A：有。騙簽時 spender 或 token 地址可以係「看起來似官方」嘅投毒地址；UI 要完整展示、校驗 checksum、同白名單對照。簽完後即使提現地址對，授權 spender 錯一樣資損。

## 小練習

**題：** 用戶投訴「冇喺交易所按提現，但錢包 USDC 被轉走」。鏈上見到：先有對 Permit2 的 `Approval`，其後 Permit2 相關 transfer 去未知 spender。交易所提現記錄為空。你會點向面試官解釋根因同平台側可做嘅防護（產品／教育／監控）？

**參考答案：**  
根因多半係 **站外騙簽 Permit2**（或 2612），唔係 CEX 熱錢包出金——所以提現單係空、但錢包有授權＋轉出事件。平台可做：(1) 安全中心展示／告警「高危授權」（spender 非白名單、無限額、長 deadline）；(2) 一鍵 revoke 指引；(3) 充值／提現 UI 永不發起模糊簽名；(4) 若產品自帶授權，嚴格 EIP-712 domain 同額度；(5) 客服 runbook：教用戶用 explorer 查 Allowance／Permit2，同地址投毒、假客服區分。面試加分：講清楚「approve(Permit2)」同「單次 permit 簽名」兩層，以及同內部出金狀態機無關、唔好誤凍錯帳。
