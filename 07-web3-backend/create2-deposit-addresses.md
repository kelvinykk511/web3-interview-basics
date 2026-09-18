# CREATE2／Factory 充值地址（CEX）

## 人話（從你熟的 CEX）

CEX 通常會畀每個用戶一個「專屬充值地址」（或一組）。鏈下你可以亂 assign 一條預先生成嘅地址；鏈上若果想 **同一套合約邏輯、可預測地址、遲啲先部署／掃事件**，成日會講 **CREATE2 + Factory**。

核心：地址唔係「隨機抽」，而係用 `keccak256(0xff ++ deployer ++ salt ++ initCodeHash)` **事先計到**。`salt` 往往綁 `userId`／`depositId`。

## 面試短答

- **CREATE**：地址依賴 nonce，難事先公開「呢個 user 嘅充值地址」。
- **CREATE2**：同一 `deployer + salt + initCode` → **確定性地址**；可先俾用戶充，再部署／用 factory 收。
- CEX 常見模式：Factory 用 `salt = hash(userId, chain, asset)` 部署（或預測）per-user receiver；`Transfer` 掃呢批地址；歸集（sweep）去 hot／cold。
- 風險：`initCode`／constructor 參數一改，地址全變；salt 碰撞／重用會共用地址 → 入帳錯戶。

## 常見追問（連答案）

**Q：點解唔全部人共用一個熱錢包地址？**  
A：共用要靠 memo／extra data（好多 ERC-20 冇）；專屬地址用 `to` 就知邊個 user，對帳簡單。CEX 仍可能對原生幣用 memo（XRP／ATOM），對 EVM ERC-20 多用 per-user 地址。

**Q：CREATE2 地址「已經存在」同「未部署」有咩分別？**  
A：未部署都可以收款（尤其 EOAs／之後先 deploy 嘅 receiver）；但有啲 token／邏輯假設 `code.length > 0`。運維要分清「地址已分配」vs「合約已部署」；掃 logs 係睇 address，唔係睇有冇 code。

**Q：同 HD wallet（BIP-44）派生有咩唔同？**  
A：HD 係從 seed 派生 **EOA 私鑰**；CREATE2 係 **合約地址公式**，私鑰喺 factory／deployer。CEX 兩者都會用：熱錢包用 HD／HSM；per-user deposit receiver 常用 CREATE2 factory。

## 小練習

**題：** 某 CEX 用 CREATE2，salt = `keccak256(userId)`。產品要求「同一 user 換一條新充值地址」（舊地址停用）。只改 salt 夠唔夠？還要注意咩？

**參考答案：**  
只改 salt（例如加 `version`／`epoch`）就可以得到新地址。注意：(1) 舊地址仍可能收到幣 → 要繼續掃或設 sweep／遷移流程，唔可以当「人間蒸發」；(2) 文檔／DB 要記錄 `address → userId + version`，禁止 salt 重用到另一個 user；(3) `initCodeHash` 唔好喺「只係想換地址」時順便改，否則整批預測地址失效。
