# Private Tx／Builder／防搶跑路徑（CEX 出金與換匯）

## 人話

公開 mempool＝廣播給所有人睇。  
**Private transaction**（經 Flashbots Protect、MEV Blocker、各 chain 嘅 private relay）＝交易先送到 **builder／中繼**，理想情況下喺入塊前唔公開暴露意圖，減少被 sandwich／搶跑。

Builder 負責組 block 交給proposer（PBS 之後更常見）。  
對 CEX：當你要做**大額鏈上 swap、敏感運維交易**，唔應該「`eth_sendRawTransaction` 丟去公共 RPC 就完」。  
類比：敏感訂單走暗池／內部通道，而唔係喊單到公開盤口等對手方夾擊。

注意：private **唔等於**絕對隱身或保證成交——relay 當機、builder 未納入、鏈唔支援時仍可能要降級；亦要信任 relay／builder 唔作惡（審查、洩漏）。

## 面試短答

Private tx 經專用 relay／builder 提交，避免意圖長時間暴露喺 public mempool，從而降低 sandwich／搶跑。  
CEX 適用：Treasury DEX 換匯、大額敏感操作；普通用戶提幣轉帳通常不必。  
取捨：隱私／防 MEV vs 額外依賴、可能延遲、成功率、供應商信任；要有公共路徑降級同監控「是否真係 private 落地」。  
加分：提到 EIP-1559 下 tip／builder 競價、同「唔好把私鑰相關運維 tx 洩去公共瀏覽器 mempool 監控」。

## 常見追問（連答案）

**Q: Private 之後係咪一定唔會被 MEV？**  
A: 唔係。Builder／搜尋者生態內仍可能有序排列與回扣；只係相對 public mempool 大幅減少「人人可見嘅經典 sandwich」。要配合滑點、拆單、限價邏輯，唔好當銀彈。

**Q: 普通 CEX 提幣要唔要用 Flashbots？**  
A: 多數唔需要——`transfer` 無價格可夾。例外：某些鏈／場景下想減少被盯上嘅「大額轉出」情報外洩，或與 swap 捆綁嘅複合交易。成本同複雜度要划算。

**Q: `eth_sendRawTransaction` 去 Infura／自建節點同送去 private relay 有何不同？**  
A: 前者通常進入該節點會傳播嘅 **public tx pool**；後者走 relay API，目標係只交畀合作 builder 組塊。接錯 endpoint＝以為 private 其實已公開。運維要配置分離同告警。

**Q: 面試點答「你們怎麼防鏈上換匯被夾」？**  
A: 分層：業務上能內盤就內盤；必須鏈上則 private submit + 嚴滑點／deadline + 拆單／TWAP + oracle 偏離熔斷；監控成交與預期價差；事故時暫停該路徑。顯示你有完整控制面，唔止背個 Flashbots 名。

## 小練習

**題：** Treasury 要喺 ETH mainnet 把 500 萬 USDC 經 DEX 換成 ETH。寫出提交路徑同失敗降級（3–5 步）。若 private relay 連續失敗，可唔可以改丟公共 mempool「快啲搞掂」？

**參考答案：**  
(1) 拆成多筆／時間切片，設每筆滑點與 deadline；(2) 簽名後經 private relay／builder 提交；(3) 監控是否入塊、成交價 vs oracle；(4) relay 失敗 → 重試備援 relay／稍後再試，**唔好**為趕時間改丟公共 mempool 一筆巨無霸（等於請人 sandwich）；(5) 超時則暫停自動換匯改人工／內盤替代，並告警。寧可延遲，唔好公開暴露大單意圖。
