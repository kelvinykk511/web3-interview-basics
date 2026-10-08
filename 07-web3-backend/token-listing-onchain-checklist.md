# 新幣上架：鏈上盡調 Checklist（CEX 錢包／帳本視角）

> 銜接：之前分開講過 [decimals](../04-smart-contracts/erc20-decimals-precision.md)、[非標準 ERC-20](../04-smart-contracts/erc20-nonstandard-quirks.md)、[proxy](../04-smart-contracts/upgradeable-proxy-uups-transparent.md)、[blacklist](./stablecoin-blacklist-freeze.md)。今日將佢哋串成「上架前後端要 check 乜」。

## 人話

上架唔只係商務同市場嘅事。對錢包後端嚟講，一個 token 合約就係「一個你控制唔到、隨時可以被改規則嘅外部系統」。上架前要答清楚三條問題：**認唔認得準（邊個合約）、入帳數字啱唔啱（餘額同 event 對唔對得上）、規則會唔會被人改（權限同升級）。**

## 面試短答（Checklist）

1. **合約地址**：最少兩個官方來源交叉確認（項目方官方文檔、官方簽名公告），explorer 上 source 要 verified。資產 key 用 `chainId + contract`，**唔好用 symbol**（假幣可以起同一個名）。
2. **多鏈／橋接版本**：同一個「USDC」喺唔同鏈係唔同合約，原生版同橋接版（例如 USDC vs USDC.e）係**唔同資產**，要分開配置、分開入帳，唔好自動合併。
3. **decimals**：上鏈讀 `decimals()`，唔好信第三方網站；寫入資產配置之後鎖死，變咗就報警。
4. **轉賬語義**：
   - fee-on-transfer：收到嘅少過 `amount`，要用 balance 前後差入帳；
   - rebasing（例如 stETH）：餘額自己變但冇 `Transfer` event，要用 balance 對帳，或者只支援 wrapped 版本（例如 wstETH）；
   - 返回值唔標準（USDT `transfer` 冇 bool）：要用 SafeERC20 式處理；
   - 有 hook（ERC-777）：要防重入。
5. **特權功能**：有冇 `mint`、`pause`、`blacklist`、可改手續費、`owner` 可以 burn／轉走別人嘅幣？owner 係 EOA 定多簽＋timelock？
6. **可升級性**：用 `eth_getStorageAt` 讀 EIP-1967 implementation slot 同 admin slot。可升級就代表上面所有結論都可以一夜之間失效。
7. **流動性／分佈**：前幾大地址佔幾多、項目方有冇大量未解鎖嘅幣。主要係風控同市場嘅判斷，但錢包要知道冷熱錢包上限點設。
8. **演練**：testnet 加 mainnet 小額跑一次全流程：充值、確認數、歸集、出金、對帳，每一步都要對得上。

## 常見追問（連答案）

**Q1：點解資產 key 唔可以用 symbol？**  
A：任何人都可以部署一個 `symbol = "USDT"` 嘅合約，向你充值地址打假幣（同 address poisoning 一個套路）。掃描器只認白名單入面嘅 `(chainId, contract)`，其他 Transfer event 一律忽略或者入隔離區。Symbol 只係顯示用。

**Q2：上架之後，鏈上邊啲 event 要持續監控？觸發咗點做？**  
A：`Upgraded(address)`（換實現）、`AdminChanged`、`OwnershipTransferred`、`Paused`／`Unpaused`、blacklist 類 event、手續費參數變更。觸發之後自動**暫停該幣充提**（fail-closed），由錢包同安全團隊重新審新實現，審完先恢復。原因：升級可以改 decimals 行為、加手續費、甚至加後門。

**Q3：rebasing token 點入帳？**  
A：兩個做法：(1) 只支援 wrapped 非 rebasing 版本（最穩陣）；(2) 真係要支援，就帳本記「份額」而唔係「數量」，或者定期用鏈上 `balanceOf` 對帳，差額歸平台或按規則分配。絕對唔可以淨係靠 Transfer event，因為 rebase 唔會 emit 每個持有人嘅 Transfer。

**Q4：fee-on-transfer 幣，用戶出金 100，對方收到 98，算唔算出金失敗？**  
A：唔算失敗，但要喺產品同帳本講清楚。出金前講明「鏈上會扣 x%」，帳本扣用戶 100，記錄實際到帳 98，差額係 token 合約收走，唔係平台收入。歸集同樣有損耗，對帳要預咗呢筆差額，否則每次都會報「對帳不平」。

**Q5：項目方話「合約係 proxy，不過 admin 係我哋 3/5 多簽」，夠唔夠？**  
A：未夠。要再問：有冇 timelock（升級前有冇時間反應）、簽名人係咪獨立嘅人／設備、admin 有冇 renounce 計劃。同時我哋自己要做 Q2 嘅監控，因為就算多簽都可以被社工或者私鑰被盜（Bybit 事件就係簽名流程出事）。

## 小練習（題目後即附參考答案）

**題：** 商務話「呢個幣下星期一上架，合約地址係 Telegram 群度畀嘅」。你作為錢包後端 owner，上架前最少要做邊 5 樣嘢？

**參考答案：**
1. 用官方網站／文檔加 explorer verified source 交叉確認合約地址，Telegram 唔算可信來源。
2. 上鏈讀 `decimals`、`symbol`、`totalSupply`，用 `getStorageAt` 讀 EIP-1967 slot 判斷係咪可升級，同埋查 admin／owner 係邊個。
3. 掃一次 ABI 同 source，睇有冇 mint／pause／blacklist／fee 開關，同埋 transfer 語義（fee-on-transfer、rebasing、返回值）。
4. 用 `(chainId, contract)` 寫資產配置；如果有多鏈版本，每條鏈獨立一條配置，原生版同橋接版分開。
5. 主網小額跑一次充值、歸集、出金、對帳，同時開咗 Upgraded／OwnershipTransferred／Paused 監控，觸發就自動暫停，跑完先開放。
