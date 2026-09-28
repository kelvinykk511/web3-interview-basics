# eth_getStorageAt 對帳（CEX 餘額／slot 校驗）

## 人話（從你熟的 CEX）

日常入帳多數靠 `Transfer` logs／`eth_getLogs`——似「掃流水」。但對帳、屍檢、可疑 token、或者非標準合約冇可靠 event 時，你要直接讀合約 storage：某個地址喺 token 合約入面嘅 `balance` slot 到底係幾多。

`eth_getStorageAt(contract, slot, block)` = 讀某個地址喺某一個 storage slot 嘅 32-byte raw 值。類比：唔經 repository API，直接 `SELECT` 某一格；你要知 schema（slot 點算），讀錯格就係錯帳。

同 `balanceOf`／`eth_call` 互補：view 方便但依賴合約 code，亦可能被怪異實作影響；raw storage 係狀態根下嘅真相（仍要懂 layout）。

## 面試短答

- **API**：`eth_getStorageAt(address, position, blockTag)` → `0x` + 64 hex（一個 word）。`latest`／具體高度／`finalized` 視對帳水位。
- **ERC-20 balance 常見 layout（Solidity 映射）**：`balances[user]` 多數喺 `keccak256(abi.encode(user, slotIndex))`，`slotIndex` 係 mapping 宣告次序（要對 bytecode／storage layout，唔好猜）。OpenZeppelin ERC-20 常見 `_balances` 喺 slot 0——**唔係標準保證**，升級代理、自研 token、packed struct 會變。
- **CEX 用途**：(1) 熱錢包／歸集地址對帳：storage 讀出 vs 內部帳本 vs `balanceOf` 三方一致；(2) 可疑充值 token：確認「真係有 balance」而唔係淨係假 Transfer log；(3) 代理合約：有時要讀 implementation 或特定 admin slot；(4) 同 `debug_storageRangeAt`／explorer storage 互補做屍檢。
- **限制**：要 Archive／夠歷史嘅節點先穩讀舊 block；唔知 layout 就唔好當對帳依據；mapping 嵌套、packed、immutable／transient 唔喺普通 storage 路徑。

## 常見追問（連答案）

**Q：已經有 balanceOf，點解仲要 getStorageAt？**  
A：`balanceOf` 係執行合約 code 嘅 view——可以被惡意／怪異實作騙（返回假數字、消耗怪 gas、甚至依賴 `msg.sender`）。對帳同風控有時要 **raw storage** 交叉驗證。正常白名單 token 日常仍用 `balanceOf`／logs；高風險上幣、事故排查先上 storage。

**Q：slot 點搵？講錯會點？**  
A：優先：編譯產物 storage layout、官方 docs、OpenZeppelin 版本對照、本地 fork 改餘額睇邊格變。錯 slot = 讀到別的變量（allowance、totalSupply、零值），對帳會「假平」或「假差」。生產腳本要把 **chainId × token × layoutVersion** 釘死，上幣審核一併收。

**Q：proxy token 讀邏輯合約 slot 定 proxy 地址？**  
A：用戶餘額通常在 **proxy 地址** 嘅 storage（delegatecall 寫入 caller storage）。對 `proxyAddress` 做 getStorageAt；讀 implementation 合約地址嘅同名 slot 多數係空／無關。Admin／implementation slot（EIP-1967）另計。

**Q：同 stateOverride 改 storage 有咩關係？**  
A：`getStorageAt` 係讀真狀態；`eth_call`+`stateDiff` 係模擬時臨時改 slot。對帳只用前者；預檢／單元測試先用後者。唔好拿 override 成功當對帳通過。

## 小練習

**題：** 白名單 USDT（假設經審核 `_balances` 在 slot 0）熱錢包對帳：內部帳本鎖後應有 1000e6，`balanceOf(hot)=1000e6`，但你用錯公式把 user 當 slot 數字直接 `getStorageAt(usdt, hotAddressAsUint)` 讀到 0。點判斷？正確 slot 點算（概念）？

**參考答案：**  
1. **唔好信「storage=0 就冇幣」**——先信已審核嘅 `balanceOf` + 近期 Transfer 淨額；同時警惕非標準 token。  
2. 錯因：mapping 唔係 `slot = address`，而係 `keccak256(pad(address) || pad(uint256(slotIndex)))`（大端 32-byte 編碼）。slotIndex=0 時對 `hot` 做呢個 hash 先係 position。  
3. 修正後 raw 值解碼成 uint256 應對齊 1000e6；若 `balanceOf` 同正確 storage 不一致 → 當異常 token／proxy／admin 可改餘額，凍結相關充值路徑並升級審核。對帳腳本要單測 layout，禁止「地址當 slot」。
