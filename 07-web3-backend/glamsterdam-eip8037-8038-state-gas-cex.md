# Glamsterdam EIP-8037／8038：狀態 gas 重定價（CEX 出金／轉帳）

> 銜接：合約收款寫死 21000／2300 見 [eth-withdrawal-to-contract-wallet.md](./eth-withdrawal-to-contract-wallet.md)，出金 bump 見 [withdrawal-gas-bump-ops.md](./withdrawal-gas-bump-ops.md)，升級熔斷見 [chain-upgrade-halt-playbook.md](./chain-upgrade-halt-playbook.md)。Blob／L2 數據費係另一條市場，見 [blob-base-fee-bpo-l2-quote.md](./blob-base-fee-bpo-l2-quote.md)；協議內 PBS（ePBS）見 [eip7732-epbs-cex-inclusion.md](./eip7732-epbs-cex-inclusion.md)。今日講 **Glamsterdam 提案點樣改 L1 狀態 gas**，以及交易所出金 bot 同 ERC-20 `transfer` 會中邊啲招。

官方說明：[EIP-8037](https://eips.ethereum.org/EIPS/eip-8037)、[EIP-8038](https://eips.ethereum.org/EIPS/eip-8038)、[EF 2026-08-24 測試影響](https://blog.ethereum.org/2026/08/24/glamsterdam-repricing-testing)、[EF 2026-11-06 徵集受影響合約](https://blog.ethereum.org/2026/11/06/glamsterdam-8037-8038)、[Hoodi 延期](https://blog.ethereum.org/2026/10/22/glamsterdam-hoodi-delay)。**數字同規格仍可能改，唔好當已定案。**

## 人話（從你熟的 CEX）

Fusaka 之後，Ethereum 下一個升級叫 **Glamsterdam**（執行層 Amsterdam ＋ 共識層 Gloas）。其中兩條提案專門「重新標價」觸碰鏈上狀態有幾貴：

- **EIP-8037**：新建狀態（新帳戶、新 storage slot、新合約 bytecode）加價，而且把呢部分同普通執行 gas **分開計量**（提案裡叫 state-gas vs execution-gas，用 reservoir 模型）。
- **EIP-8038**：讀／寫已有狀態亦加價，例如冷帳戶訪問、`SSTORE` 寫入附加費。

用出金系統嘅講法：你以前可能寫死「ERC-20 出金 65,000 gas 一定夠」，或者對合約用 `transfer()` 只留 2300 stipend。分叉之後，**第一次**轉某隻幣去一個從未持有過佢嘅地址，等於幫對方新建 storage slot，成本會明顯高過「轉俾已經有餘額嘅地址」。寫死 gas 嘅出金會 Out of Gas；靠 `gasleft()` 計費或者硬編碼 stipend 嘅合約，甚至可能加 limit 都救唔到。

重點係時間線：截至 **2026-11-17**，Glamsterdam **未上主網**，Hoodi 測試網分叉已延期，EIP 自己都寫參數 **not yet final**。EF 11-06／11-09 開咗受影響不變量徵集，審計報告交俾 ACD，**11-19 ACDE** 先討論發現同可能改規格，**冇預先承諾決議**。所以而家要做嘅係盤點同準備，唔係當主網已生效去改生產參數。

## 面試短答

1. **未上主網，規格可能改。** 答題先講：呢個係 Fusaka 之後嘅 L1 升級提案，不是已生效規則。實際數字以最終 EIP／客户端為準。CEX 而家要做回歸清單同測試網演練，唔好提前把主網出金 gas 當成「已經加價」。
2. **新建狀態同執行分開計，而且會貴好多。** 新帳戶、新 slot、新 bytecode 走 state-gas；普通執行走 execution-gas。用戶仍然只設一個 `tx.gas`，但 `gasleft()`／`GAS` 未必反映全部可用嚟建狀態嘅預算（reservoir）。後台用「開頭減結尾 gas」去計費會錯。
3. **首次收款 ≠ 舊地址轉帳。** 出金到從未持有該 ERC-20 嘅地址，token 合約要 `SSTORE` 一個新 slot；首次轉 ETH 去全新帳戶亦有新帳戶成本。冷帳戶訪問喺 8038 亦會加價。gas 估算必須用廣播當下嘅 `eth_estimateGas` ＋ buffer，唔好用歷史平均值或寫死常數。
4. **硬編碼 gas 係第一失效模式。** EF 回放歷史交易嘅結論大致分四類：冇變化、成功但 gas 不同、加高 limit 就過、即使大幅加 limit 都可能壞。後者典型係：Solidity `transfer`／`send` 嘅 2300 stipend、合約或 relayer 寫死 gas、按 `gasleft()` 分支／計費、預簽死 gas limit 嘅交易。EF 明確話審計覆蓋唔到所有**鏈下**代碼——CEX 出金 signer、批量派發、預簽佇列正好屬呢類。
5. **要有分叉當日 playbook。** 上架代幣同自研 payout 合約要自查；分叉前後對「首次收款」「合約收款」做回歸。若某隻已上架代幣 `transfer` 開始確定性失敗，暫停該幣充提，唔好當 RPC 故障無限重試。主網日期未定，接到官方測試網再跑，唔好估日。

## 常見追問（連答案）

**Q1：同 blob／BPO 費有咩分別？出金手續費要唔要一齊改？**  
A：唔同一條市場。Blob 費（4844／Fusaka／BPO）係 L2 發布數據嘅價格，影響 L2 出金成本。8037／8038 改嘅係 **L1 執行同狀態** 嘅 gas 表，影響主網 ETH／ERC-20 出金、歸集、以及任何會新建狀態嘅合約調用。兩邊都要動態報價，但熔斷訊號同監控指標要分開，唔好用 L1 execution gas 去代替 blob，亦唔好用 blob 去估主網 `transfer`。

**Q2：出金到「已經有 ETH 嘅舊用戶」係咪就唔使理？**  
A：唔係。佢可以早已有 ETH，但第一次收呢隻 ERC-20 仍然係新 slot。冷／熱訪問、對方係合約（`receive`／hook）都會令 `estimateGas` 同實際消耗唔同。正確做法係每筆（至少按「新／舊 token 餘額」分檔）現估，並對首次收款留更大 buffer。

**Q3：分叉日未定，面試官問「而家做咩」點答？**  
A：三件實事：(1) 清出金／歸集路徑上所有寫死 gas、2300 stipend、`gasleft()` 計費；(2) 列出可能受影響嘅已上架代幣同自研合約，接上 chain-upgrade 暫停流程；(3) 等官方測試網／最終規格再回歸，睇 ACDE 會唔會改 EIP。唔好把提案數字寫進生產常數。

## 小練習（題目 + 參考答案）

**題目：** 出金服務把所有 ERC-20 轉帳寫死 `gasLimit = 65,000`，批量 ETH 派發合約用 `payable(to).transfer(amt)`。Glamsterdam 主網上線後第一日：多筆「新地址首次收 USDT」出金 `status=0`（OOG），另有一隻已上架代幣嘅內部結算開始大面積 revert。你點診斷同改？

**參考答案：**

- **先睇 receipt**：OOG 同業務 revert 要分開。65,000 唔夠通常係首次 `SSTORE`／冷訪問加價；加 limit 都過唔到，就要懷疑對方合約寫死 stipend 或 `gasleft()` 邏輯，係「代幣可能壞咗」而唔只係你估少咗。
- **出金 bot**：刪寫死 65,000，改為簽名前 `eth_estimateGas` ＋ buffer ＋ 單筆上限；對「收款地址該 token 餘額為 0」單獨加大 buffer。失敗入「鏈上失敗」狀態，OOG 可調高後重試，確定性 revert 要暫停該幣並人工看合約。
- **批量 `transfer()`**：同 10-28 課一樣改 `call` ＋ 單筆 gas cap ＋ 失敗唔拖死成批。2300 stipend 喺重定價後更易爆。
- **預簽佇列**：已簽死 gas 嘅交易唔好喺分叉後直接廣播；過期／作廢再按新規則重簽。
- **Playbook**：主網激活日前喺官方測試網用「新地址首次收款」同「舊地址」兩組案例回歸；激活後監控該幣失敗率，必要時停充提。記住規格可能仲會改，常數要可配置。
