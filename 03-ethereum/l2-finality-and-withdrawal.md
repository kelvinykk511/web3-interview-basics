# L2 最終性與提款延遲（CEX 視角）

## 人話

CEX 後端對 L1（例如 Ethereum）習慣「等 N 個 confirmation 就當最終」。  
L2（Optimism / Arbitrum / Base 等）不一樣：

- **入金（L2 → CEX）**：多數 optimistic / rollup 上，交易幾秒就「看起來確認」，但真正不可逆要等 L2 自己的最終性規則（同有冇可能被 sequencer / challenge 影響）。
- **出金（L2 → L1 再到用戶）**：很多 L2 從 L2 提回 L1 有 **challenge / dispute window**（常見 7 日量級），呢段時間資產喺「橋接途中」，唔係即時到 L1。

對 CEX：**入帳確認數**同**用戶提幣到 L1 嘅 SLA** 要分開設計，唔好當「鏈上轉帳就即到」。

## 面試短答

Rollup L2 把執行放到二層、把資料／證明提交到 L1。  
Optimistic rollup：先假設正確，有挑戰期；期間從 L2 提回 L1 要等挑戰窗結束。  
ZK rollup：靠有效性證明，提回 L1 通常快過 optimistic，但仍受證明產生／L1 打包影響。  
CEX 處理 L2 資產時：入金用 L2 finality／安全確認策略；跨層提款要標明延遲同狀態機（pending bridge → L1 credited），唔好同 L1 原生轉帳混同一套 confirmation 常數。

## 常見追問（連答案）

**Q: 點解 Arbitrum／Optimism 提回 Ethereum 要等好耐？**  
A: Optimistic rollup 靠「有人發現錯就挑戰」。提款要等 challenge period 結束先可以喺 L1 釋放資金，防止錯誤狀態被偷提。呢段係協議安全設計，唔係單純「網絡塞車」。

**Q: 咁 CEX 列出 USDT-Arbitrum，用戶提「到 ERC-20 Ethereum」係咪即到？**  
A: 唔一定。若果熱錢包只喺 Arbitrum，要跨到 Ethereum L1，往往要行官方橋或流動性橋／內部調撥。官方橋有延遲；CEX 多數用自有多鏈熱錢包＋內部 ledger 對沖，對外顯示「處理中」，背後做跨鏈調撥。面試要講清：**產品鏈** vs **內部資金調撥路徑** 係兩回事。

**Q: L2 入金要等幾多 confirmation？**  
A: 視風險模型。Sequencer 軟確認可以好快，但 CEX 通常仲會等 L2 block 深度或「已 post 到 L1」某種程度，再入帳。原則同 L1：可接受 reorg／欺詐風險 vs 入帳速度嘅權衡；L2 多一層 sequencer／challenge 風險，唔好照抄 ETH=12 確認。

**Q: Soft finality vs hard finality？**  
A: Soft：sequencer／L2 節點話「已打包」，UX 快但可被重組或挑戰。Hard：狀態根已喺 L1 最終確認（或 ZK proof 已驗證），逆轉成本極高。CEX 大額入帳傾向等更硬嘅最終性。

## 小練習

**題：** 用戶喺 CEX 提「ETH 到 Arbitrum 地址」vs「ETH 到 Ethereum 地址」，後端狀態機有咩關鍵差別？

**參考答案：**  
- 到 Arbitrum：熱錢包若已有 Arbitrum ETH，就係一筆 L2 `sendRawTransaction`，跟 L2 nonce／gas／finality；狀態大致 `created → broadcast → l2_confirmed → done`。  
- 到 Ethereum：若資金主要喺 L2，可能要先 L2→L1 bridge（長延遲）或從 L1 熱錢包直接出；狀態要多 `bridging`／`l1_pending`，SLA 同客服文案都唔同。  
核心：目標鏈決定簽名、RPC、確認規則；跨層仲要 bridge 子狀態，唔好共用同一個 `confirmations=12`。
