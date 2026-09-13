# Multicall 批次讀鏈（CEX 後端視角）

## 人話

CEX 後端成日要查：一堆地址嘅 ETH balance、某 ERC-20 餘額、allowance、nonce……  
若果對每個地址各打一次 `eth_call`／`eth_getBalance`，等於 N 次 RPC round-trip——慢、貴、仲容易撞 rate limit。

**Multicall**（常見係 `Multicall3` 合約）把多個「唯讀 call」包成**一筆** `eth_call`：合約喺同一個 block 內依序執行，一次返回全部結果。  
類比你後端：唔好 for-loop 打 100 次下游 HTTP，改成一次 batch API；而且結果係**同一區塊視圖**，避免「半途區塊前進」導致餘額不一致。

注意：Multicall 主要係**讀**（simulate／view）。真正轉帳、approve 仍然要各自簽名廣播，唔係「一筆 tx 代替所有寫入」嘅萬能藥（除非你自己寫 batch 合約並承擔權限風險）。

## 面試短答

Multicall 用鏈上 aggregator 合約，將多個 target + calldata 合成一次 `eth_call`，在**單一 block** 拿到一致快照。  
CEX 場景：掃熱錢包餘額、批量查 token balance／allowance、組裝出金前檢查。  
好處：少 RPC、降延遲、一致視圖；壞處：單一 call gas／response size 有上限，某個 subcall revert 要處理（Multicall3 可選 `allowFailure`），唔能取代寫入路徑嘅簽名與 nonce 管理。

## 常見追問（連答案）

**Q: Multicall 同 JSON-RPC batch（一個 HTTP 塞多個 eth_call）有咩分別？**  
A: RPC batch 只係網絡層打包，每個 call 仍可能喺唔同瞬間／唔同 block 執行，節點實作亦可能部分支援。Multicall 合約保證**同一 block 上下文**執行全部 subcall，適合要「同一快照」嘅對帳。生產上兩者可併用：RPC batch 減少 HTTP；Multicall 保證一致性。

**Q: 某個 subcall revert，成條 Multicall 會唔會掛？****  
A: 舊版／嚴格模式可能整筆失敗。Multicall3 常用 `aggregate3` + `allowFailure=true`：失敗 subcall 回 `success=false`，其餘繼續。CEX 對帳應逐項檢查 success，唔好假設全部有數。

**Q: 可唔可以用 Multicall「一次過」做完所有出金？**  
A: 唔建議把多用戶出金塞進一個有金鑰權限嘅 batch 寫入合約——攻擊面同 blast radius 極大。出金仍應每筆（或嚴格設計嘅內部 batcher）獨立簽名、獨立 nonce／狀態機。Multicall 留給**讀路徑**同極少數受控嘅運維合約。

**Q: 點解面試愛問 Multicall？**  
A: 考你知唔知「讀鏈成本」同「一致快照」。CEX 掃鏈／風控／熱錢包監控若無 batch，RPC 帳單同延遲會爆；答到 block 一致性同 allowFailure，就同純背 API 名嘅人分開。

## 小練習

**題：** 風控要喺同一個區塊視圖核對：10 個熱錢包嘅 ETH、某 USDT 餘額、同對 router 嘅 allowance。用 Multicall 點設計？若其中 1 個 `balanceOf` 因地址對錯 revert，你希望成條失敗定局部失敗？

**參考答案：**  
組 Multicall3 `aggregate3`：對每個地址各加 `getEthBalance`（或多 call 原生 balance helper）、USDT `balanceOf`、USDT `allowance(wallet, router)`。全部 `allowFailure=true`（或至少對可能壞嘅 call 開），逐項睇 `success`；錯地址嗰項記告警，其餘 9 個錢包仍用**同一 block** 結果入風控快照。唔好用「一失敗就整批作廢」除非你要嚴格 all-or-nothing。
