# USDT0 全鏈 USDT：上架、充提同金庫調撥

> 銜接：OFT 機制、eid／guid／DVN、同 CCTP 對比見 [layerzero-oft-vs-cctp-rebalancing.md](./layerzero-oft-vs-cctp-rebalancing.md)。今日落到 **USDT0 呢個產品**：經典 Ethereum USDT、各鏈 USDT0、同 Legacy Mesh 接住嘅 Tron／TON 原生 USDT，係唔同合約、唔同路徑，交易所最容易用 ticker 混帳。

## 人話（從你熟的 CEX）

用戶口裡「USDT」其實可能係三樣嘢：Ethereum 上 Tether 經典合約、某 L2／新鏈上嘅 **USDT0**、以及 Tron／TON 等鏈上一直存在、並唔係 USDT0 嘅原生 USDT。USDT0 用 LayerZero OFT 把流動性收成一張網：喺 Ethereum 把經典 USDT **鎖入官方 OFT Adapter**，其他 OFT 鏈 **mint USDT0**；之後呢啲鏈之間互轉係 burn／mint，唔再郁 Ethereum 鎖倉。另外仲有 **Legacy Mesh**，專門接「只有舊版原生 USDT、唔係 USDT0」嘅鏈，走池子 lock／unlock，**唔係 1:1 全額**（官方文件寫緊約 **0.03%** fee）。

好似你哋帳本要用 `(chainId, 合約)` 做資產 key，唔可以因為 ticker 都叫 USDT 就入同一條目——呢個同 USDC vs USDC.e 同一類錯。

## 面試短答

1. **資產要拆開**。Ethereum 經典 USDT（`0xdAC17F958D2ee523a2206206994597C13D831ec7`）通常經官方 **OFT Adapter** 鎖倉後，先喺其他鏈出現 USDT0；目標鏈入帳嘅係 **USDT0 合約**，唔係嗰條鏈上可能仲存在嘅舊橋 USDT。合約地址以 [USDT0 Deployments](https://docs.usdt0.to/technical-documentation/deployments) 為準（Ethereum Adapter 示例：`0x6C96dE32CEa08842dcc4058c14d3aaAD7Fa41dee`，上線前再核對官方頁）。
2. **兩條路唔好當成同一條。** **OFT Mesh**（已部署 USDT0 嘅鏈之間）：burn／mint，原則上 **1:1**（仍要處理 dust／報價）。從其他鏈 **redeem 返 Ethereum**＝burn USDT0，Adapter **unlock** 經典 USDT，適合金庫把貨調返主鏈。**Legacy Mesh**（官方現時提到 Tron、TON 等原生 USDT，經 Arbitrum hub 池子）：credit／debit 池餘額，**收 fee**，有額度上限；唔可以當「send 幾多到帳幾多」。部分專用 lockbox（例如 IOTA）只同 Ethereum 配對，**唔能直達**其他 USDT0 鏈，要先返 Ethereum。
3. **安全棧要核對、亦要監控變更。** 官方開發文件寫緊 USDT0 用 **3/3 DVN**（LayerZero、USDT0、Canary），三個都要過先執行。呢個係當前公開配置，**唔好寫死當永遠不變**；運行期對 LayerZero Scan／OApp config，admin 改 peer、DVN、確認數要當變更事件，必要時暫停充提。
4. **充提掃描**。只 credit 已上架嘅官方合約。用戶把 USDT0 打去「只收該鏈舊 USDT／經典 USDT」嘅充值地址，走錯幣找回，**唔好當 USDT 入帳**。出金要問清楚用戶要嘅係 USDT0 還是該鏈原生 USDT，熱錢包分池，唔好「出 USDT 去 Arbitrum」就隨便揀一個合約。
5. **金庫調撥**。官方 USDT0 路徑用 OFT `send` 狀態機（見 OFT 篇），一張單一次 send，用 guid 補執行。走 Legacy Mesh 要按 `quoteOFT` 計 **扣 fee 後到帳**，庫存先入在途再按實際 credit 入帳。USDT0 可以 redeem 成 Ethereum 原生 USDT，減少自己再經第三方橋。

## 常見追問（連答案）

**Q1：USDT0 係咪「官方 USDT」？同 USDC 嘅 CCTP 一樣？**  
A：USDT0 係 Tether／USDT0 網絡把經典 USDT 鎖倉後，喺其他鏈按 OFT 鑄出嚟嘅官方全鏈版本，**1:1 由 Ethereum 鎖倉支撐**（可公開對鎖倉）。但擔保模型係 **OFT + 已公布 DVN 集合**，唔係 Circle attestation，亦唔係每條鏈上 Tether 自己再發一隻互相唔互通嘅原生 USDT。USDC 跨鏈優先仍用 CCTP。

**Q2：Tron USDT 可唔可以當 USDT0 直接調去 Base？**  
A：Tron 上常見嘅係原生 USDT，走嘅係 **Legacy Mesh**（經 Arbitrum hub），有 fee、靠池子流動性，唔係 OFT Mesh 嗰種 1:1 burn/mint。要入 OFT 網之後先可以按 USDT0 路徑去 Base。Job 唔可以假設到帳＝發送額，必須 quote，並且分清楚資產科目。

**Q3：Scanner 只監 Transfer，會唔會漏？**  
A：用戶直接轉 USDT0 到充值地址，有普通 `Transfer`，同收其他 ERC-20 一樣。但若你只 allowlist 咗舊橋合約，USDT0 轉入會被當成未知代幣。金庫自己做 OFT `send` 時，源鏈先有 debit，目標要等 `OFTReceived`／mint 嘅 Transfer 先入庫存。guid 用來把兩端事件對上，防重複入帳。

## 小練習（題目 + 參考答案）

**題目：** 產品要支持「Arbitrum USDT」充值，scanner 只訂閱咗市場上較早出現嘅一隻 USDT 合約。同時金庫 job 用 Legacy Mesh 把 200 萬 Tron 原生 USDT 調去 Base，狀態機按 1:1 先把 200 萬入 Base 可用庫存，再到帳時發現少咗一截。有咩問題？點改？

**參考答案：**

- **充值漏入帳／入錯帳**：Arbitrum 而家用戶好可能存入嘅係 **USDT0**（另一張官方合約）。只掃舊合約＝漏單，或者有人存舊橋 USDT 你當可互換 USDT 入帳，之後贖唔到官方路徑。正確係 `(chainId, contract)` 分開上架；USDT0 用官方 Deployments 地址；舊橋資產要獨立政策（拒收、或另行 naming）。
- **Legacy Mesh 唔係 1:1**：Tron→Base 經 Legacy Mesh 會扣約 0.03% fee，仲受池子限額影響。未到帳就按 200 萬記入 Base 可用，等於把未到手、而且本應較少嘅金額當成可出金。應先入「跨鏈在途」，`quoteOFT` 預留 fee，目標實際 credit 幾多先入庫存；差額入費用科目。
- **路徑要對網**：若目的係 Base 上嘅 USDT0 庫存，確認官方路徑（Legacy 再入 OFT Mesh，定係已有嘅多跳），用 guid／官方瀏覽器對帳，失敗只補執行唔重發。IOTA 類專用 lockbox 唔好當可任意直達。
- **回歸**：用官方 USDT0 合約打一筆進測試充值地址必須入帳；用舊橋合約要拒收或入另一個科目。Legacy 調撥對帳金額必須等於實際到帳而唔係發送額。
