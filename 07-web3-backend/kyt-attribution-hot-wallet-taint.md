# KYT 深化：對手歸因、合約／橋／他所得充值、熱錢包污染

> 銜接：KYT 基本流程、超時 fail-closed、冪等、事後告警、dusting、tipping-off 見 [kyt-onchain-risk-screening.md](./kyt-onchain-risk-screening.md)；穩定幣合約層凍結見 [stablecoin-blacklist-freeze.md](./stablecoin-blacklist-freeze.md)；出金 Travel Rule 見 [travel-rule-withdrawal-vasp.md](./travel-rule-withdrawal-vasp.md)。今日專注 **「from 顯示邊個」唔等於「錢真正從邊個經濟來源嚟」**，同埋 **歸集之後污染點傳播**。

## 人話（從你熟的 CEX）

KYC 係認識個客；基礎 KYT 係對一條地址／一筆 tx 攞風險分。實際充值好多時 `Transfer.from` 唔係用戶自己嘅 EOA，而係：

- 另一間交易所嘅熱錢包（用戶「由幣安提過來」）
- 官方橋／第三方橋合約（用戶喺另一條鏈 lock／burn，呢邊 mint／unlock）
- DEX router、聚合器、支付合約
- Mixer／privacy pool／未驗證 proxy

供應商如果只睇「上一跳」，橋同大所熱錢包通常分數好低——因為嗰個地址本身係正當實體。但錢喺上一跳之前可能先經過混幣器。呢個好似反洗錢入面嘅 **層層轉帳**：你唔可以只問「匯款行係邊間」，仲要問「資金來源係邊度」。

另一面：CEX 自己嘅 sweep 會把好多用戶充值地址歸集入幾條熱錢包。對手 VASP 同 KYT 供應商會用聚類（clustering）把你嘅熱錢包標成「某間交易所」。一旦有一筆高風險資金 sweep 入主熱錢包，**之後由呢條熱錢包出金嘅其他用戶**，都可能被對手當成「嚟自受污染集群」而 hold。物理上錢已經混咗；帳本層就算分開記，鏈上睇唔出。

## 面試短答

KYT 唔可以只對 receipt 上嘅 `from`／`Transfer.from` 打分。要分清 **直接對手（counterparty）** 同 **經濟來源（origin）**：合約、橋、他所得熱錢包屬於「通道」，真正風險往往喺更早嘅 hop 或者另一條鏈。工程上：(1) 識別 `from` 實體類型（EOA／已知 VASP／橋／DEX／未標記合約）；(2) 通道類要升級做 **多跳／跨鏈追溯**（供應商 cross-chain tracing，或者自己對 origin chain 再查）；(3) 智能合約入金必要時用 `debug_trace`／internal transfer 拆出真正付款人，唔好把 router 當成用戶；(4) **未完成篩查或者 HELD 嘅資金，唔好 sweep 入主熱錢包**，應入隔離地址；(5) 熱錢包一旦混入高風險資金，後續出金可能被下游 VASP 擋，要有錢包分段、污染標記同必要時輪換地址。結果仍然唔可以向用戶透露內部風險標籤。

## 常見追問（連答案）

**Q1：點解「from 係 Circle CCTP／官方橋」低風險，仍然可能要 hold？**  
A：橋合約係通道，唔係來源。用戶可以喺來源鏈用污染地址 burn／lock，再喺你支援嘅鏈收 wrapped／native 鑄造額。單鏈 KYT 只見到「橋 → 你嘅充值地址」。要嘛用有跨鏈追溯能力嘅供應商（跟 mint 對應嘅 source tx），要嘛對大額另做來源鏈核對。唔好因為橋本身喺白名單就自動 `CREDITED`。

**Q2：用戶從另一間 CEX 提幣入嚟，KYT 只能見到對方熱錢包，點處理？**  
A：這係常態，對方熱錢包作為 VASP 實體通常唔會直接標成 mixer。合規策略多數係：視對方 VASP 風險評級＋金額＋用戶本身 KYC 層級決定要唔要額外來源證明；同時做出金側 Travel Rule（若對方亦係 VASP）。工程上要維護「已知 VASP 熱錢包」名單（供應商 entity attribution），同普通 EOA 分開規則，而唔係一律當低風險直入。

**Q3：`tx.to` 係用戶充值地址，但 ERC-20 `Transfer.from` 係一個 router，點歸因？**  
A：入帳仍然以 **Token Transfer 到你派俾呢個 userId 嘅地址** 為準（金額、合約、logIndex）。KYT 嘅對手則要用 Transfer.from；若 from 係合約，再查 internal call／上一跳，避免把 Uniswap Router 標成「用戶來源」。冪等鍵仍然係 `(chain, txHash, logIndex)`，同一筆 multicall 入面可以有多個用戶入帳，唔好用 `tx.from` 當唯一客戶。

**Q4：已經 sweep 入主熱錢包先發現來源係被盜資金，點？**  
A：帳本層對該筆做補凍結／HELD（見基礎篇事後告警），並標記「主熱錢包喺時間 T 起有污染暴露」。鏈上層：視合規要求決定係咪停止再用這條熱錢包做出金、把未污染餘額遷去新熱錢包、定係繼續用但接受下游拒絕率上升。重點係 **事先用隔離地址避免混倉**：`SCREENING`／`HELD` 未放行前，sweep 路由去 quarantine，而唔係 treasury hot wallet。兩個系統（歸集 vs 合規）要讀同一狀態。

**Q5：出金目標係未驗證 proxy、mixer router 或者隱私池，簽唔簽名？**  
A：出金 KYT 除咗篩 EOA，仲要睇目標實體類型。制裁／mixer／明顯混淆協議通常 **fail-closed 拒簽**；未知合約可以按金額升級人工，而唔係當普通地址。用戶話「呢個係我自己嘅 Smart Account」時，要走地址所有權驗證／白名單（見 [withdrawal-address-ownership-verification.md](./withdrawal-address-ownership-verification.md) 同 [eth-withdrawal-to-contract-wallet.md](./eth-withdrawal-to-contract-wallet.md)），唔可以只靠「合約有 code 所以拒」或者「冇 code 所以準」。

**Q6：對手 VASP 話「你哋熱錢包嚟嘅錢有風險」，但你內部帳本顯示嗰個用戶好乾淨，點解釋？**  
A：鏈上聚類睇嘅係 **你熱錢包呢個實體**，唔係你內部 userId。你 sweep 把所有人嘅錢混埋，下游只見到「交易所熱錢包 → 用戶」。所以隔離同錢包分段（例如合規隔離池 vs 零售出金熱錢包 vs 機構專用）係工程控制，唔單止係合規政策。對外仍然唔好 tipping-off；對內要能回答「呢條熱錢包近期吸收過邊幾筆 HELD／高風險歸集」。

## 小練習（題目 + 參考答案）

**題目：** 一筆 80,000 USDC 經某條官方橋合約打入用戶充值地址。即時 KYT 對 `Transfer.from`（橋地址）回「已知橋、低風險」。sweep 服務預設確認數一夠就歸集去主熱錢包。兩日後供應商跨鏈追溯顯示：來源鏈對應嘅 lock 來自混幣器。同時開始有其他用戶投訴「提去另一間交易所被 hold」。你會點設計，避免再犯？而家已經混咗又點善後？

**參考答案：**

- **事前（正確設計）**：識別 `from` 為橋／VASP／router 就升級追溯，唔用「通道低風險」直接 `CREDITED`。大額可以先 `SCREENING`（跨鏈結果返回或逾時轉人工）。sweep 讀 KYT 狀態：未放行或 HELD 只准進入 **隔離地址**，禁止入主熱錢包。
- **歸因**：入帳金額仍以本鏈 Transfer 為準；KYT 檔案要存直接對手（橋）＋來源鏈 origin tx／地址＋供應商報告版本同時間，方便事後重評。
- **善後**：對該 80,000 補凍結／合規工單；標記主熱錢包喺 sweep 當刻起有暴露；評估係暫停該熱錢包出金、遷移乾淨餘額到新地址，定係只對大額出金加嚴。調查近期由該熱錢包出金被下游 hold 嘅單，提供內部對帳（唔向用戶透露「你收到mixer錢」）。
- **對外**：用戶若問點解 hold，只講需要額外審核／補來源證明，API 唔暴露風險類別。
- **回歸**：橋＋低風險通道但缺 origin → 唔自動入帳；隔離地址歸集唔入主熱；EOA 直轉且低風險先走快速入帳；出金到已標記 mixer router 拒簽。
