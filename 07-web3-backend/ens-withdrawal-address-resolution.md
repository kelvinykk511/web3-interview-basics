# ENS 出金：用戶填 `vitalik.eth` 點處理

> 2026-11-04 warm-up。CEX 出金頁越來越多人想填域名（ENS、`cb.id` 子域名等）而唔係 0x 地址。

## 人話
ENS 好似 DNS：`kelvin.eth` 只係一個「指標」，真正收錢嘅係佢而家指向嘅 0x 地址。
指標隨時可以被 owner 改、域名過期之後仲可以俾第三者註冊。
所以後台鐵律：**域名只係輸入方式，出金單、白名單、簽名永遠鎖死解析出嚟嗰個 0x 地址。**

## 解析流程（以太坊主網）
1. 名稱正規化（ENSIP-15 normalize）——擋大小寫、全形、homoglyph（例如西里爾字母 `а` 扮 `a`）。正規化失敗直接拒絕。
2. `namehash(name)` 得到 node。
3. 問 ENS Registry：`resolver(node)` → resolver 合約地址；冇 resolver 就係解析唔到。
4. 問 resolver：`addr(node)`（ETH）；其他鏈用 ENSIP-9 `addr(node, coinType)`，EVM 鏈 coinType = `0x80000000 | chainId`（ENSIP-11）。
5. Wildcard／offchain resolver（ENSIP-10 + CCIP-Read EIP-3668）：resolver 會 revert `OffchainLookup`，叫你去 HTTP gateway 攞答案再驗簽。好多子域名（例如交易所／錢包派嘅）係咁做。
   現成 library（ethers / viem / web3j 嘅 ENS 支援）都會處理呢步，自己寫好易漏。

## 後台設計要點
- **解析一次、俾用戶確認 0x、鎖入訂單**：出金單存 `input_name` + `resolved_address` + 解析 block number。之後風控、KYT、簽名全部用 `resolved_address`，唔好簽名前再解析一次（中間被改就會出去另一個地址）。
- **白名單存地址唔存名**：域名過期（有 90 日 grace period，之後任何人可以註冊）→ 新 owner 改 record，如果白名單存名就會出錯人。
- **跨鏈唔好亂用主網 addr**：用戶喺 Arbitrum 出金，要問該鏈 coinType 嘅 record；冇就拒絕或者明確提示。智能合約錢包（Safe）喺唔同鏈地址可能唔同／未部署。
- **反向解析（address → name）只做顯示**：reverse record 任何人都可以 claim 任何名，一定要 forward 再解析一次，結果等返原地址先顯示個名。
- **地址投毒／仿冒名**：`kelvin.eth` vs `kelv1n.eth`，UI 要大字顯示解析出嘅地址頭尾，首次出金地址照走冷靜期／額外驗證。
- RPC 失敗、gateway timeout → 出金單直接失敗要用戶重試，唔好 fallback 去 cache 嘅舊結果。

## 面試短答
> ENS 只係輸入層：我哋喺主網經 Registry → resolver 解析（支援 ENSIP-10/CCIP-Read 同 per-chain coinType），正規化防 homoglyph，解析結果俾用戶確認之後同解析 block 一齊鎖入出金單；白名單、KYT、簽名都只用 0x 地址，因為 ENS record 可改、域名可過期再被註冊。反向解析只用嚟顯示，而且要 forward 驗證。

## 常見追問（連答案）
- **Q：點解唔喺簽名嗰刻先解析，咁最新？**
  A：最新唔等於用戶確認過。用戶確認嘅係某個地址，中間 record 被改（被盜 key 或者惡意 owner）你就出咗去用戶冇睇過嘅地址。鎖死確認嗰個先係可審計。
- **Q：用戶加咗 `abc.eth` 入白名單，半年後域名過期被人搶註點算？**
  A：所以白名單一開始就要存解析出嚟嘅地址（名只係備註）。即使顯示名，都要定期 forward 解析，唔一致就標記「名稱已變更」唔再顯示名。
- **Q：L2 出金點解析？**
  A：ENS 本身喺 L1，用 ENSIP-11 coinType 問 resolver 攞該鏈地址；冇 per-chain record 就唔好默認用 ETH 主網 addr（EOA 通常一樣，但合約錢包可能唔一樣），要提示或者拒絕。

## 小練習（題目 + 參考答案）
**題目：** 設計 `withdrawal_order` 入面同 ENS 有關嘅欄位，並寫出由提交到廣播邊幾步會用到邊個欄位。

**參考答案：**
- 欄位：`input_type`（ADDRESS / ENS）、`input_name`、`resolved_address`、`resolve_chain_id`、`resolve_block_number`、`resolver_address`、`user_confirmed_at`。
- 提交：後端解析 → 回傳 `resolved_address` 俾前端顯示 → 用戶確認後寫 `user_confirmed_at`。
- 風控／KYT／白名單比對：只用 `resolved_address`。
- 簽名／廣播：`to = resolved_address`，完全唔再接觸 `input_name`。
- 客服查單：顯示 `input_name` + 解析 block，可以證明「當時」個名指去邊。
