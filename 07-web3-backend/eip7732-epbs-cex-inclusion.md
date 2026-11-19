# EIP-7732：協議內 PBS（ePBS）對 CEX 充提／質押入帳

> 銜接：出金廣播同 pending 見 [hot-wallet-pending-monitor.md](./hot-wallet-pending-monitor.md)、[withdrawal-gas-bump-ops.md](./withdrawal-gas-bump-ops.md)；validator／非交易 ETH 入帳見 [eth-non-tx-balance-changes.md](./eth-non-tx-balance-changes.md)、[eip7002-el-triggerable-withdrawals.md](./eip7002-el-triggerable-withdrawals.md)；Glamsterdam 另外兩塊見 [glamsterdam-eip8037-8038-state-gas-cex.md](./glamsterdam-eip8037-8038-state-gas-cex.md)、[eip7708-eth-transfer-logs-cex.md](./eip7708-eth-transfer-logs-cex.md)。今日講 **協議點樣把 proposer／builder 分工寫入共識**，以及 CEX 掃描器同質押入帳會中邊啲招。

官方說明：[EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)（頁面標 Last Call，deadline 2026-11-01）、[ethereum.org Glamsterdam](https://ethereum.org/roadmap/glamsterdam/)（同 EIP-7928 BALs 並列 headliner，Scheduled for Inclusion）。**截至 2026-11-19 未上主網**，主網日期未定。Hoodi 因 8037／8038 已延期，唔好再引用過期嘅「2026-10-27 Hoodi」時間表。2026-11-19 ACDE #249（14:00 UTC，香港時間 22:00）主要討論 8037／8038，唔等於今天會拍板 ePBS 或宣布主網日。規格同列入名單都以 EIP 同 ACD 為準，唔好當主網已生效。

## 人話（從你熟的 CEX）

你而家知道 Ethereum 出塊其實多數已經分家：被選中嘅 proposer 唔自己排交易，而係經 MEV-Boost／relay 呢啲**協議外**中間人，向 builder 攞一個「承諾嘅 execution payload」，再交俾 relay 填返完整塊。CEX 出金、私有交易通道，本質上都係喺呢個外包市場入面搶 inclusion；relay 掛或者 builder 審查，出金就會停。

EIP-7732（ePBS）把呢種分工寫入協議：beacon block **不再直接帶** `ExecutionPayload`，改放 builder 簽過嘅 bid——承諾稍後透露嘅 execution `blockhash`、execution requests root、同要俾 proposer 嘅價錢。Builder 係一類**新嘅質押實體**（最低 1 ETH），唔做 attest／propose、唔計入 total active balance，經 EL 兩個新系統合約註冊／加注／退出，再以 EIP-7685 request type `0x03`（deposit）／`0x04`（exit）交去 CL。Payload Timeliness Committee（PTC）只證明「有冇準時透露正確 blockhash、blob 資料喺佢哋眼中可唔可用」，**唔執行、唔驗證** payload。真正嘅 execution 驗證延後：規格寫明，slot N 被 builder 打包嘅交易，要等 N+1 嘅 proposer 疊喺呢塊上面、而且 N+1 嘅 attester 投票認頭，先算廣泛驗證。

一個 slot 可以係三種：

- **Full**：beacon block 同 execution payload 都上咗鏈
- **Skipped**：連 beacon block 都冇
- **Empty**：有 beacon block，但承諾嘅 payload 冇出現

Empty 嘅時候，CL 已經扣咗嗰輪 withdrawals，但 EL 要等之後有 payload 先入帳；未履行完之前，CL **唔會再處理**下一輪 withdrawal。Builder 俾 proposer 嘅錢會變成 EL withdrawal（排喺其他 withdrawal 之前，入 proposer 指定地址）：payload 上到鏈就付；若 payload 永遠唔出，但該 beacon block 喺嗰 slot 收到至少 **60%** per-slot active balance 嘅 attestation，會喺 epoch transition 仍然付（unconditional payment）。

對 CEX：ePBS **唔等於**出金從此有審查抵抗——強制 inclusion list（FOCIL，EIP-7805）已改去後續升級 Hegotá，唔在 Glamsterdam。打包權仍在 builder。協議亦容許繼續用協議外 relay 做尚未入核嘅功能，唔好當 MEV-Boost 會即日消失。Builder 仲有「自由選擇權」：透露 payload 變得唔划算時可能扣住，造成 empty slot。

## 面試短答

1. **7732 係 Glamsterdam headliner，不是主網現況。** 截至 2026-11-19 主網啟用日未定。答「而家已經協議內 PBS、唔再需要 relay」會被當場打臉。而家做節點升級盤點同掃描器回歸，啟用當日先切行為假設。
2. **見到 beacon block ≠ 見到執行交易。** Empty slot 冇 execution payload，亦冇新嘅用戶交易 receipt。掃鏈、確認數、把「slot／beacon 頭前進」當成「又入咗一塊執行塊」會錯數。要分清共識頭同執行頭。
3. **「已入塊」唔等於已被廣泛驗證。** Slot N 嘅 tx，廣泛驗證在 N+1。大額仍然等你哋既定嘅 `safe`／`finalized`；寫死「latest + 1 個 confirmation 就入可用」會更危險。ePBS 唔改最終性定義，但把 execution 驗證移離熱路徑。
4. **質押同非交易 ETH 入帳會變形。** Builder payment 以 withdrawal 形式出現喺 proposer 指定地址；empty slot 會拖住 EL 側嘅 validator withdrawal 入帳，甚至卡住後續 sweep。呢啲都**唔係**用戶充值，走 [非交易入帳](./eth-non-tx-balance-changes.md) 獨立來源。
5. **出金審查抵抗冇因為 ePBS 自動解決。** 打包仍然靠 builder；FOCIL 不在今次升級。廣播路徑、builder API、審查／延遲告警仍然要留。唔好把「協議內公平交換」講成「一定會被打包」。

## 常見追問（連答案）

**Q1：ePBS 之後係咪唔使 MEV-Boost／relay？出金是不是更難被審查？**  
A：核心嘅「payload 換付款」變成協議內、信任最小化嘅公平交換，誠實 proposer 唔再依賴 relay 先攞到錢。但 ethereum.org 寫明，複雜功能仍可能用協議外 middleware。審查抵抗要靠 inclusion list 一類機制；FOCIL 已改去 Hegotá，所以 ePBS 本身**唔保證**你嘅出金一定進入下一個 payload。

**Q2：Empty slot 對充值掃描同質押出金有咩影響？**  
A：掃描器如果只跟 beacon／slot 號，會以為「塊已經出咗」但其實冇新 execution。用戶交易唔存在於 empty slot，唔好憑空入帳。質押側：CL 已扣 withdrawal，EL 要等下一個履行呢批 withdrawal 嘅 payload 先見到餘額；期間後續 withdrawal 會暫停。對帳要以 EL 實際入帳為準，唔好單憑 CL 已扣就當熱錢包已收到。

**Q3：Builder 最低 1 ETH、又唔係 validator，會唔會影響現有質押產品？**  
A：Builder 有獨立 registry 同 index，唔行普通 validator 嘅 churn／隊列，stake 唔計入 total active balance。CEX 若自己或合作方做 builder，要走新系統合約同 `0x03`／`0x04` request，唔好當普通存款合約。Builder 付款同已退出 builder 嘅 sweep 會以 flagged withdrawal 出現喺 EL，監控同歸集規則要認呢類來源，避免當成不明入金。

## 小練習（題目 + 參考答案）

**題目：** 主網宣布 ePBS 已啟動。充值掃描改為「beacon 頭每前進一步就當 +1 confirmation」；latest 見到 receipt 就入可用 ETH。質押監控見到 proposer 地址無 user tx 都突然多咗 ETH，當作未知用戶充值。出金團隊認為「已經協議內 PBS，可以關私有發送同審查告警」。有咩問題？點改？

**參考答案：**

- **確認數要跟執行頭，唔係只跟 beacon／slot：** empty slot 會令「確認數」虛增，但其實冇新 execution。仍以包含該 tx 嘅 execution block 起計，大額等到 `safe`／`finalized`。
- **latest receipt 更唔夠：** 規格已說明 slot N 嘅交易要到 N+1 先被廣泛驗證。ePBS 之下「見到 latest」比而家更弱，禁止用嚟入可用。
- **Builder payment／延遲 withdrawal 唔係充值：** proposer 指定地址突然多 ETH，優先當協議內付款或延遲到帳嘅 CL withdrawal，走非交易入帳唯一鍵；用戶充值仍只認明確來源。
- **唔好關審查同私有發送：** ePBS 唔附帶 FOCIL，builder 仍可唔打包、甚至扣住 payload。保留 builder／mempool 雙路徑、超時 bump、以及 inclusion 延遲告警。
- **上線節奏：** 主網日期未定就唔好當已生效；先喺測試網驗證 empty vs full、withdrawal 延遲同新 request type，分叉當日先切假設。
