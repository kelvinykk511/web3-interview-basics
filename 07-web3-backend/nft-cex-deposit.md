# ERC-721／1155 與 CEX 充值掃描

## 人話

ERC-20 充值：掃 `Transfer(from,to,value)`，`to` 係你充值地址就入帳。  
**NFT** 唔同：ERC-721 每張係唯一 `tokenId`；ERC-1155 同一合約可以有多個 `id`，一次可以轉多個數量（semi-fungible）。

CEX 若收 NFT／遊戲資產／RWA NFT：唔能夠再用「只認合約＋數量」——要認 **合約地址 × tokenId（× amount）**，同埋事件係 `Transfer`／`TransferSingle`／`TransferBatch`。  
類比帳本：ERC-20 係「帳戶餘額」；721 係「資產序號表」；1155 係「多 SKU 庫存」。

## 面試短答

- **ERC-721**：`Transfer(from,to,tokenId)`；所有權 `ownerOf(tokenId)`；入帳鍵通常係 `(chainId, contract, tokenId)`。  
- **ERC-1155**：`TransferSingle`／`TransferBatch`；餘額 `balanceOf(account,id)`；入帳鍵 `(chainId, contract, id)`，數量可 >1。  
- CEX 掃描：訂閱／輪詢對應事件，`to` ∈ 充值地址集合先入帳；要處理 **safeTransfer** 回調、批量、同假事件／錯標準合約。  
- 風險：惡意 NFT（奇怪 metadata／回調重入）、用戶轉錯合約、同一視覺系列多個假合約、提現要對應正確 `tokenId`。  
面試想聽：你分得清 20／721／1155 事件同入帳主鍵，同埋 NFT 充提唔係「加個 decimals」咁簡單。

## 常見追問（連答案）

**Q: 點解唔好用 `balanceOf` 輪詢代替掃事件？**  
A: 721 嘅 `balanceOf(address)` 只知「有幾多張」，唔知係邊啲 `tokenId`；要枚舉通常靠事件索引或 `tokenOfOwnerByIndex`（唔係人人都實作 Enumerable）。1155 更要知邊個 `id`。生產入帳仍以 **事件＋最終性** 為主，balance 只做對帳。

**Q: 用戶把 NFT safeTransfer 到合約地址會點？**  
A: 若收款方係合約且冇實作 `onERC721Received`／`onERC1155Received`，safe 路徑會 revert；唔 safe 嘅 `transferFrom` 可能令 NFT 卡死合約。CEX 充值地址多數用 **EOA**（熱錢包地址）收，避呢類坑；若用合約收，必須實作 receiver 並當成安全審計項。

**Q: 假充值／釣魚 NFT 點防？**  
A: 只白名單（chainId + 合約地址）；唔好只認 name／symbol／圖片。入帳後 metadata 可展示，但 **帳務真相係合約＋tokenId**。提現只允許提出已入帳嘅同一三元組。

**Q: 同 ERC-20 充值掃描共用一套流水線？**  
A: 可共用「掃鏈 → 確認數 → 入帳狀態機」，但 parser／主鍵／對帳／提現組 tx（`transferFrom` vs `safeTransferFrom` vs 1155 `safeBatchTransferFrom`）要分開。唔好硬套 `amount` 當唯一欄位。

## 小練習

**題：** 用戶聲稱已把某系列 NFT `#42` 充去 CEX 展示地址，鏈上你見到一筆 ERC-721 `Transfer` 去咗正確地址，但合約地址唔喺白名單（山寨合約、同名 metadata）。系統應否入帳？面試點答？

**參考答案：**  
**唔入帳。** 帳務只認白名單 `(chainId, contract)`；山寨合約嘅 `#42` 同藍籌 `#42` 係兩種資產。應顯示「未支援／未知合約」類拒絕，而唔係按圖片入帳。加分：講清楚提現時亦只會從托管錢包轉出已登記嘅真合約 tokenId，避免「入假提真」。
