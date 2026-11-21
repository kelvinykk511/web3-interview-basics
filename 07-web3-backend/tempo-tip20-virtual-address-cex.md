# Tempo：穩定幣手續費同虛擬充值地址（CEX 上架／充提）

> 銜接：MegaETH 仍然用 ETH 付 gas；Tempo 連呢樣都唔一樣。CEX 習以為常嘅「每客一個充值地址 + 定時 sweep」同「熱錢包備原生 gas 幣」，喺 Tempo 嘅 TIP-20 路上要改模型。對照 [CREATE2 充值地址](./create2-deposit-addresses.md)、[熱錢包 sweep](./hot-wallet-sweep-consolidation.md)、[穩定幣黑名單](./stablecoin-blacklist-freeze.md)。

官方（以而家文件為準）：[Network details](https://tempo.xyz/developers/docs/quickstart/connection-details)、[Fees](https://tempo.xyz/developers/docs/network/fees)、[pathUSD / quote tokens](https://tempo.xyz/developers/docs/protocol/exchange/quote-tokens)、[Predeployed contracts](https://tempo.xyz/developers/docs/quickstart/predeployed-contracts)、[Virtual addresses](https://tempo.xyz/developers/docs/protocol/tip20/virtual-addresses)、[Customer deposit addresses](https://tempo.xyz/developers/docs/guide/payments/virtual-addresses)、[Consensus and finality](https://tempo.xyz/developers/docs/protocol/blockspace/consensus)、[T3 upgrade](https://tempo.xyz/developers/docs/protocol/upgrades/t3)、[Evaluate stablecoin settlement](https://tempo.xyz/developers/docs/resources/guides/stablecoin-settlement)。截至 2026-11-21。

## 人話（從你熟的 CEX）

Tempo 係一條為穩定幣支付而設嘅 L1（主網 chainId **4217**，測試網 Moderato **42431**）。RPC `https://rpc.tempo.xyz`，貨幣符號顯示 **USD**。最重要嘅一句：**冇原生 gas 代幣。** 手續費用合資格、以 USD 計價嘅 **TIP-20** 穩定幣付；未指定時，後備費用代幣係 **pathUSD** `0x20c0000000000000000000000000000000000000`（6 decimals），同時亦係官方 DEX 嘅根報價代幣。錢包可能顯示一個「原生餘額」占位，文件叫你唔好當真正資金——用戶同熱錢包嘅錢喺 TIP-20 餘額度。

出金交易係特製類型（文件示例 `type: "0x76"`，帶 `feeToken`）。可以自己付費，亦可以另一個 fee payer 代付。Gas price 以 **attodollar**（10⁻¹⁸ USD）每單位 gas 計，再換算成最多 6 位小數嘅 USD。第一次轉去未持有該代幣嘅地址，gas 可以比轉去舊持有人高，唔好對所有出金寫死同一個 limit。Fee Manager 在 `0xfeec000000000000000000000000000000000000`；所選 fee token 要喺 Fee AMM 有足夠流動性，先換到驗證者想收嘅幣。

共識係 **Simplex BFT**（Commonware）。正常情況出塊約 **600ms**（500ms builder loop 再加網絡同驗證）。最終性係確定性而唔係概率性：塊被標成 finalized 之後**唔可以回滾**，文件寫明確「no reorg risk after finality」。整合方用 `eth_getBlockByNumber("finalized")`，**最終性之後唔使再加確認數**。驗證者集合而家係許可制；安全優先於活性——超過三分之一驗證者離線時網絡會停，而唔會出衝突塊。所以「出塊快」仍然要跟 `finalized`；網絡停擺時要有暫停充提 playbook。

充值側有一個專門為交易所而設嘅能力：**虛擬地址**（T3，主網 **2026-04-27 16:00 CEST** 已生效，約香港時間 22:00）。你註冊一個 master 錢包，之後離線為每個客戶派生充值地址，格式係：

`masterId`（4 bytes）＋ 固定 `VIRTUAL_MAGIC`（10 bytes）＋ `userTag`（6 bytes）

客戶把 **TIP-20** 轉去呢個地址時，預編譯喺**同一筆** transfer 入面查 registry，直接把餘額記入 master。虛擬地址本身 **`balanceOf` 永遠係 0**，唔需要再 sweep。同一筆交易會出兩條 `Transfer`：

1. `Transfer(sender → virtual, amount)`（用於歸因，從地址尾 6 bytes 解 `userTag`）
2. `Transfer(virtual → master, amount)`（真正入庫）

如果你嘅掃描器把每條 `Transfer` 當一筆獨立充值，就會**重複入帳**，而且收款人會認錯。mint／帶 memo 嘅路徑仲有更多事件，但都要摺成「一筆邏輯充值到 master」。若 sender 同 master 係同一個地址，仍然會出兩條 Transfer，但淨餘額變化係 0，對帳時要排除。

政策唔會因為虛擬地址而放水：TIP-403 檢查嘅係 **resolved master**，master 唔准收，虛擬地址條路一樣失敗。Master 嘅 receive policy 如果擋住交付，資金會去 **ReceivePolicyGuard** `0xB10C000000000000000000000000000000000000`，而唔係 master——入帳前要確認錢真係到咗 master，而唔係被扣住。虛擬轉發**只適用於 TIP-20 嘅 transfer／mint**；普通 ERC-20 轉去虛擬地址會當成普通 20-byte 地址，**唔會**自動轉去 master，幣可以卡喺嗰個「冇人當錢包用」嘅地址。

## 面試短答

1. **Tempo 冇原生 gas，熱錢包要備合資格 TIP-20（例如 pathUSD），唔好備「原生 ETH／USD」。** 錢包占位原生餘額唔能當用戶資金或手續費來源。出金帶 `feeToken`，用實際 estimate，新收款地址可能更貴。
2. **入帳等 `finalized`，之後唔使再疊確認數。** Simplex BFT 確定性最終性，出塊約 600ms；停擺（超過 1/3 驗證者離線）要當鏈不可用，而唔係用 confirmation 硬撐。
3. **TIP-20 虛擬充值地址唔持有餘額。** `balanceOf(virtual)` 永遠 0；歸因靠事件同 `userTag`，餘額只喺 master。
4. **兩條 `Transfer` 摺成一筆邏輯入帳。** 唔好掃一次入一次。入帳前確認 master 餘額真係增加，而唔係被 receive policy 扣喺 Guard。
5. **只有 TIP-20 會轉發。** 非 TIP-20 嘅 ERC-20、或者會按字面地址記 ownership 嘅協議，唔好派虛擬地址去收。

## 常見追問（連答案）

**Q1：點解掃描 `balanceOf(充值地址)` 永遠係 0，客戶卻話已轉帳？**  
A：虛擬地址只係路由別名。錢喺同一筆 TIP-20 調用入面已經記入 master。要核對 `Transfer(sender → virtual)` 同 `Transfer(virtual → master)`，再用 `userTag` 對客；同時確認唔喺 ReceivePolicyGuard。唔好因為 balance 係 0 就當未到，亦唔好對虛擬地址做 sweep。

**Q2：可唔可以沿用「每客一個 EOA、定時歸集」？**  
A：TIP-20 路上唔需要，亦唔應該再為每個虛擬地址付一筆 sweep。但若你收嘅係普通 ERC-20（非 TIP-20），虛擬轉發唔生效，舊模型先派得上。上架時要分清該資產係咪 TIP-20。Fee 仍然要從 master／fee payer 嘅穩定幣出，記住備 fee token。

**Q3：`latest` 見到 receipt 可唔可以即入可用？**  
A：官方叫整合方把 **finalized** 當作不可逆結算，最終性之後唔使再加確認。快確認同 finalized 唔係同一件事。另外最終性只保證「呢筆交易唔會被踢出正典」，仲要核對正確網絡、正確合約、正確收款人同金額；被 policy 扣住唔算已交付。

## 小練習（題目 + 參考答案）

**題目：** Tempo 上架穩定幣充提。產品沿用 EVM 模板：認為 gas 係原生代幣，熱錢包只監控「原生餘額」；每客派一個虛擬充值地址。掃描器對每條 `Transfer` 入一次帳，又用 `balanceOf(虛擬地址) > 0` 做到帳確認。`latest` receipt 即入可用。有用戶把一隻非 TIP-20 嘅 ERC-20 轉去虛擬地址。另有一筆 TIP-20 其實被 receive policy 扣咗去 Guard，但第一條 `Transfer(sender → virtual)` 已經觸發入帳。有咩問題？點改？

**參考答案：**

- **手續費同餘額讀 TIP-20：** 冇原生 gas。熱錢包備 pathUSD（或已驗證有 Fee AMM 流動性嘅 fee token），出金用 `0x76`／`feeToken` 做 estimate。忽略錢包占位原生餘額。
- **入帳對齊 `finalized`，並把兩跳 Transfer 摺成一筆到 master。** 用 `txHash + log` 組合成唯一鍵時，要認「成組事件」而唔係每條 log 入一次。`balanceOf(virtual)` 不能做到帳條件。
- **確認 master 實收：** 入帳前看 master 餘額增加（或等價嘅成功交付證據）。若資金喺 ReceivePolicyGuard `0xB10C…0000`，唔能當用戶已入可用。
- **虛擬地址只收 TIP-20：** 非 TIP-20 轉去虛擬地址唔會轉發，要當錯資產／可能卡住，走人工找回，而唔好假設協議會幫你 sweep。對外充值說明寫清只支持已上架 TIP-20。
