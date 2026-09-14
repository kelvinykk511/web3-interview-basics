# MEV／三明治攻擊（CEX 後端視角）

## 人話

鏈上 mempool 多數公開：你廣播一筆「用 ETH 換 USDC」或者「大額 swap」未上鏈之前，搜尋者已經睇到。  
**MEV（Maximal Extractable Value）**＝排序／插入／審查交易可以賺到嘅額外價值。  
最常見用戶痛點係 **sandwich（三明治）**：搜尋者喺你前面買貴、你成交後再賣，你滑點變差，差價入咗佢袋。

CEX 後端多數出入金係「轉帳到固定地址」，唔係公開 DEX 市價單——**單純 ERC-20 `transfer` 入冷熱錢包通常唔會被 sandwich**。  
但一旦你做：**鏈上換匯、Treasury 再平衡、跨池搬倉、用戶「一鍵兌換後提幣」**，就踏入同 DeFi trader 一樣嘅 MEV 戰場。  
類比撮合引擎：公開 order book 被人搶價；你而家係把意圖晾喺公開 mempool。

## 面試短答

MEV 係區塊生產者／搜尋者透過交易排序與插入獲取嘅價值；sandwich 係典型形式（front-run + back-run 夾擊受害交易）。  
對 CEX：普通充提轉帳風險低；**鏈上 swap／再平衡／聚合路由**高風險。  
防護：設嚴滑點與 deadline、拆單、用 private relay／builder（唔公開進 public mempool）、避免可預測大單、監控異常成交價。  
面試想聽：你分得清「轉帳」同「可被套利嘅意圖交易」，同埋 CEX 內部鏈上操作要當高風險路徑設計。

## 常見追問（連答案）

**Q: 用戶從 CEX 提 ETH 去自己錢包，會唔會被 sandwich？**  
A: 單純 `transfer`／原生轉帳通常**唔會**——無價格曲線畀人夾。風險多在：用戶隨後自己去 DEX 換、或者 CEX 產品把「提幣前自動 swap」做成一筆公開意圖交易。

**Q: CEX 熱錢包要不要理 MEV？**  
A: 要，視操作類型。純出金轉帳：重點係簽名、nonce、最終性、地址污染；**Treasury 用 DEX／aggregator 換穩定幣／再平衡**就要當 MEV 目標：用 private flow、限價／TWAP、多池拆單，唔好一筆巨無霸公開 swap。

**Q: Sandwich 同「普通搶跑買同一 token」差在哪？**  
A: 搶跑可以只係睇到你大買而跟買；sandwich 係**前後夾擊同一筆受害者交易**，直接吃你滑點。答題時講清 victim tx 被夾在中間。

**Q: 點解同 CEX 撮合有得比？**  
A: CEX 撮合喺私有撮合引擎，順序由你控制；鏈上公開 mempool 等於把意圖廣播給全世界搜尋者。所以「CEX 內盤成交」同「鏈上公開 swap」威脅模型完全唔同——後者要額外防 MEV。

## 小練習

**題：** 產品要做「用戶用帳戶 USDT 一鍵換成 ETH 再提到外址」。兩種實作：(A) CEX 內盤撮合完再鏈上轉 ETH；(B) 後端直接丟一筆公開 DEX swap 再轉。MEV／風控上你揀邊個？點講面試？

**參考答案：**  
優先 (A)：價格在 CEX 內盤形成，鏈上只做普通轉帳，避開 public mempool sandwich。若必須 (B)：用 private relay／builder、嚴滑點＋deadline、限制單筆／頻率，並監控成交價偏離 oracle；面試強調「意圖唔好晾喺公開 mempool」。
