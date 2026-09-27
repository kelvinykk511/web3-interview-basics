# 熱錢包 Pending Tx 監控（CEX Ops／風控）

## 人話（從你熟的 CEX）

熱錢包係「會自動簽 tx 出金／歸集」嘅熱路徑。出問題唔淨係「鏈上已確認少咗錢」——仲有一大類：**mempool 裏面已經有（或應該有）pending 交易**，你哋狀態機唔知、或者多咗唔認得嘅 pending。

類比：支付核心有「處理中訂單」表；你要對齊「渠道側未決單」。鏈上用 `nonce` + pending／latest 計數，再配 `eth_getTransactionByHash`／（可選）txpool，對齊業務單。

呢條同「卡住提現點 RBF」（mempool-and-rbf）互補：RBF 講點救一筆；呢度講點**持續監察熱錢包係咪健康、有冇被盜簽、有冇 nonce 卡死**。

## 面試短答

- **兩個計數**：`eth_getTransactionCount(addr, "latest")` = 下一個可用於「已確認」嘅 nonce；`"pending"` = 計入 mempool 未決後嘅下一 nonce。`pending - latest` ≈ 該地址未決 tx 數量（節點視角）。
- **健康信號**：出金服務廣播後，對應 `txHash` 應能喺 pending 或之後 receipt 見到；業務單狀態 `BROADCAST` 要綁呢個 hash。
- **告警**：
  - `pendingCount` 長期 > 閾值 → 可能 fee 過低／nonce 洞／RPC mempool 不同步。
  - **出現你哋系統無記錄嘅 from=熱錢包 pending** → 極高優先：可能金鑰洩漏或另一套腳本在簽。
  - `latest` 前進但對應唔到已知出金單 → 對帳／安全事件。
- **多節點**：不同 RPC 嘅 mempool 視圖唔一致；監控要「法定節點／私有節點」為準，告警用多數或主節點，避免抖動。
- **L2／私有池**：有啲鏈／私有 tx 路徑 mempool 查唔到；要以 builder 回執 + 自建「已簽名未確認」表為準，唔好假設一定有 `txpool_content`。

## 常見追問（連答案）

**Q：`pending` 同 `latest` 一樣，係咪代表冇風險？**  
A：只代表呢個 RPC 睇唔到未決。可能 tx 已上鏈（剛確認）、或廣播失敗、或走咗 private relay。仍要對業務表：有冇 `BROADCAST` 超時無 receipt；熱錢包餘額／nonce 要同帳本對。

**Q：點樣發現「被盜用熱鑰發出嘅 pending」？**  
A：(1) 訂閱／輪詢該地址 pending nonce 空隙同 tx；(2) 每筆 from=熱錢包嘅 pending／mined tx 必須能關聯出金／歸集單號；(3) 無關聯 → 凍結出金、轉冷／輪換地址、撤銷授權（ERC-20 approve）、事故響應。單靠餘額告警會遲一步。

**Q：同 stuck-nonce／RBF 點分工？**  
A：監控負責**發現**（pending 堆積、未知 tx、nonce 洞）。RBF／cancel 負責**處置**已知業務單（同 nonce 加速或自轉取消）。未知 pending **唔好**隨便 RBF 成你哋嘅提現——可能同攻擊者搶 nonce；應先安全響應再決定 replace／放棄。

**Q：HD 充值地址要唔要同樣監控 pending？**  
A：充值地址通常唔向外簽（除 sweep）。監控重點係「sweep 熱鑰／歸集地址」同「出金熱錢包」。充值地址出現 outbound pending 同樣係紅燈。

## 小練習

**題：** 出金熱錢包 `latest=105`、`pending=108`。業務庫只有兩筆 `BROADCAST`（nonce 105、106）。RPC 仲見到一筆 nonce=107、to=陌生地址、value 好大嘅 pending。你做邊三步？可唔可以立刻用 nonce 107 發一筆更高 fee 嘅「取消」？

**參考答案：**  
1. **當安全事件**：暫停該熱錢包新簽名；頁面／Pager 告警；保留 raw tx／RPC 證據。  
2. **核對**：105／106 是否你哋單；107 是否任何合法歸集／手動運維（通常唔係）→ 當未授權。  
3. **處置選項**：若你仍控制鑰且策略允許，可用**同 nonce 107** 更高 fee 打取消（to=自己、value=0）或打去安全地址搶上鏈——呢係同攻擊者競速，要運維 runbook，唔係普通提現 RBF。同時輪換熱錢包、掃 approve、冷錢包撥備。  
唔可以「當普通卡住提現」自動對 107 做業務 RBF 成用戶提現；nonce 已被未知用途佔用。105／106 繼續按 receipt 跟進；整條出金切備用熱錢包。
