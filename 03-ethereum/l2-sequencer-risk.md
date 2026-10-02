# L2 Sequencer 風險與中心化（CEX 視角）

## 人話

好多 L2（Optimism、Arbitrum、Base…）而家交易預設經 **sequencer** 排序打包。  
人話：有個「專責出塊／排序」嘅營運方，先給你軟確認，再把資料／狀態根交去 L1。

對 CEX：
- **好處**：確認快、費低、UX 似 Web2。
- **風險**：sequencer 掛咗 → 入金／出金卡住；排序不公 → 用戶／內部交易被插隊；極端時審查／停機。  
呢啲唔係「區塊鏈永遠唔會停」嘅行銷句——L2 有明確嘅**營運依賴**。類比 CEX：撮合引擎掛咗，撮合單唔會 magically 自己成交——你要有降級同狀態機，唔好當「廣播成功＝完成」。

昨日／前幾日講 soft vs hard finality、getProof／withdrawal；今日補：**軟確認背後係邊個喺度排序**，同 CEX 入出金狀態點掛鉤。

## 面試短答

多數 rollup 早期用單一／少數 sequencer 提供快速軟確認與有效吞吐。  
風險包括：可用性（宕機）、審查（拒收某 tx）、MEV／排序偏袒、同升級金鑰／治理中心化。  
CEX 應對：入帳唔好只信 sequencer soft confirm——大額等更硬最終性或 L1 沉降；出金有降級路徑（換 RPC、等強制包含／逃生艙若協議支援）；產品與風控要為「L2 暫停出塊」準備狀態機同客服話術；多鏈分散熱錢包流動性，避免單 L2 單點。

## 常見追問（連答案）

**Q: Sequencer 掛咗，用戶係咪永遠提唔到幣？**  
A: 視協議。理想設計有 **force inclusion / escape hatch**：逾時後可經 L1 強制提交交易或退出。實務上用戶體驗差、延遲長，CEX 要當「罕見但要 rehearsal 嘅事故」。唔好假設「永遠有即時 sequencer」。

**Q: Soft confirm 同「已上 L1」差喺邊？**  
A: Soft：sequencer 話收咗／排咗，快但依賴營運方誠實同在線。已 post 到 L1（data／state root）：逆轉成本靠近 L1 安全。CEX 大額入帳應定義清楚用邊一級；細額可以快入帳＋風控限額。

**Q: Sequencer 審查（censorship）對 CEX 出金有咩實務影響？**  
A: 若 sequencer 拒收／延遲特定地址或合約呼叫，熱錢包出金可能長時間 `pending` 卻唔上鏈。監控要睇：同 nonce 喺 mempool／sequencer 佇列停留過久、官方狀態頁、同業出金是否同步卡住。應對：暫停該鏈出金、標記 `sequencer_degraded`、必要時走協議強制包含／換路徑（若有），唔好無限重播同一 tip 當「gas 問題」。

**Q: CEX 列出某 L2 代幣，運維上要監控咩？**  
A: Sequencer／批次提交是否正常、L1 上 rollup 合約是否在推進、最終性延遲有無拉長、官方狀態頁／緊急公告；出金隊列積壓同熱錢包該鏈餘額。事故時：暫停該鏈入出金、ledger 標記 `chain_halted`，避免以為「廣播成功」就完成。

## 小練習

**題：** 熱錢包對某 L2 連續廣播 3 筆出金，explorer 長時間顯示 pending；L2 gas oracle 正常，L1 blob fee 亦平。同業 Telegram 開始傳「該鏈 sequencer 抽風」。你出金狀態機下一步點做？

**參考答案：**  
先當 **sequencer／排序可用性** 問題，唔好當 tip 唔夠一路加價重播。動作：（1）暫停該鏈新出金入隊；（2）已廣播未確認標記 `sequencer_uncertain`，唔寫「已完成」；（3）核對官方狀態頁／L1 批次是否仲推進；（4）細額已 soft 入帳嘅充值提高凍結門檻。恢復後先對帳 nonce／實際上鏈結果再放行，並 rehearsal 一次 force-inclusion／客服話術。
