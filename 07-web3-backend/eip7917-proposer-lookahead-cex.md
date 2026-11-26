# EIP-7917：確定性 proposer lookahead 對 CEX 出金／預確認

> 銜接：你已經識 [EIP-7251 MaxEB／consolidation](./eip7251-validator-consolidation.md)、[EIP-7732 ePBS](./eip7732-epbs-cex-inclusion.md)、[Fast Confirmation Rule](./fast-confirmation-rule-cex.md)、同 [EIP-7805 FOCIL](./eip7805-focil-cex-inclusion.md)。EIP-7917 解決嘅係「邊個會出未來呢幾格塊」可唔可以提前鎖死，**唔係**審查抵抗，亦**唔係**最終性。同另外三樣好易混。

官方（以 2026-11-26 文件為準）：[EIP-7917](https://eips.ethereum.org/EIPS/eip-7917)、[Fulu beacon-chain 規格](https://ethereum.github.io/consensus-specs/specs/fulu/beacon-chain/)、[EIP-7607 Fusaka Meta](https://eips.ethereum.org/EIPS/eip-7607)、[ethereum.org Fusaka](https://ethereum.org/roadmap/fusaka/)（該頁 2026-10-16 更新）。Fusaka 已於 **2025-12-03 21:49:11 UTC**（香港時間 2025-12-04 05:49:11）主網生效，epoch **411392**，fork ID `0x5167e2a6`。呢篇係生產知識，唔係未上線路線圖。

## 人話（從你熟的 CEX）

CEX 出金最想知「呢筆何時被打包」。以前共識層雖然提早知道 RANDAO 種子，但 **epoch N+1 嘅 proposer 名單喺 epoch N 期間並未完全鎖死**：有效餘額（effective balance）喺 epoch 內可以變——罰沒、懲罰，以及 Pectra／EIP-7251 之後嘅存款、合併、最高 2048 ETH 複利，都可能令「下一輪邊個出塊」同你先前計嘅唔同。

EIP-7917 嘅做法好直接：每個 epoch 邊界，提前算好一份名單，寫入 beacon state 嘅 `proposer_lookahead`。呢個欄位係一條 validator index 向量，長度 `(MIN_SEED_LOOKAHEAD + 1) × SLOTS_PER_EPOCH`。而家 `MIN_SEED_LOOKAHEAD = 1`、`SLOTS_PER_EPOCH = 32`，即 **64 個 slot**——涵蓋**當前 epoch 加下一個 epoch**。`proposer_lookahead[0]` 係本 epoch 第一格嘅出塊人，`proposer_lookahead[32 + 4]` 係下一個 epoch 第五格，如此類推。`get_beacon_proposer_index` 改為讀 `proposer_lookahead[state.slot % 32]`，因為過咗 epoch 邊界會先把舊 epoch 整段移走，下標 0–31 永遠對準**當前** epoch。

用內部系統比喻：唔係即時搶單，而係把「未來兩班（兩個 epoch）邊個值班」寫入一張不可中途改動嘅排班表。從本 epoch 開頭計，呢個窗口係 64 × 12 秒 ＝ **約 12.8 分鐘**；你喺 epoch 中段望出去，剩下可見嘅未來值班大約係 6.4 至 12.8 分鐘。過邊界時移走已結束嗰個 epoch，再補上再下一個 epoch 嘅名單。

但「知道邊個值班」**唔等於**「你嘅出金一定會被佢入塊」。預確認（preconfirmation）係另外同呢個已知 proposer 做嘅承諾，用嚟加速 UX；協議只保證名單穩定、可證明，唔保證收你嘅交易。大額入帳、不可逆出金，仍然要等塊真係出咗，再跟你哋自己嘅確認政策（多數係 `finalized`，或者已啟用嘅 FCR `safe` 政策）。唔好把 lookahead 當成加急通道。

點解要而家先講：Fusaka 已經上線將近一年。基於預確認（based preconf）可以靠這份穩定名單運作，合約亦可以透過 beacon block root（EIP-4788）加 Merkle proof 驗證「某一個未來 slot 係邊個 validator」。CEX 若自己跑驗證者，監控「下一 epoch 職責」亦唔再會因為中途 EB 變動而改卦。

## 面試短答

1. **EIP-7917 已隨 Fusaka 於 2025-12-03 21:49:11 UTC 上主網。** 它把未來 proposer 名單預先寫入 `BeaconState.proposer_lookahead`，喺當前 epoch 期間唔會再改。
2. **窗口係當前 epoch + 下一 epoch，共 64 個 slot（約 12.8 分鐘，從本 epoch 開頭計）。** `MIN_SEED_LOOKAHEAD = 1`。過 epoch 邊界先滑動補上再下一個 epoch。
3. **名單確定 ≠ 交易會被包容。** 預確認係同已知 proposer 嘅額外承諾，唔係協議保證，更加唔係最終性。
4. **同 FCR／FOCIL／ePBS 分工唔同。** 7917 只回答「邊個會出塊」；FCR 係已出塊嘅快速確認；FOCIL（Hegotá，未上主網）逼下一塊收公開交易；ePBS（Glamsterdam，未上主網）改 payload 生產同披露。
5. **EIP-7251 令 EB 更容易變，先突顯舊設計嘅不確定性。** 7917 同時封咗「見到 RANDAO 先改 EB 去影響下一 epoch 出塊權」呢條路。出金 SLA 同入帳政策唔好綁住某一個未來 slot。

## 常見追問（連答案）

**Q1：而家可以唔可以根據 lookahead，把出金超時改成「等下一個已知 proposer 嗰格」？**  
A：唔可以當協議保證。你只係知道邊個會出嗰格塊，唔知道佢會唔會收你、塊會唔會缺席、交易係咪仍有效。超時仍然要按 mempool 可見性、是否已入塊、確認深度分開處理。把 3 分鐘超時改成「12 秒一格就重播」，好容易卡 nonce。

**Q2：用戶或對手拿出 preconf，話下一個 slot 必定入塊，可唔可以提早入可用？**  
A：唔可以當最終性。Preconf 係對某個已知 proposer 嘅承諾，信任假設喺嗰個驗證者／協議，唔喺以太坊經濟最終性。CEX 大額、出金對沖、不可逆操作仍要等實際區塊，再跟 `finalized` 或已審批嘅確認政策。Lookahead 只令呢類承諾嘅「對象」穩定，唔係替你背書。

**Q3：同 FCR、FOCIL、ePBS 點分？會唔會 Fusaka 已經一齊帶來 FOCIL？**  
A：唔會。Fusaka 帶嘅係 7917（同 PeerDAS 等），唔包 FOCIL 或 ePBS。FCR 改嘅係已出塊嘅 `safe` 語義；FOCIL 用包容名單約束下一格；ePBS 改 builder／payload 披露。四樣解決四個問題。面試最忌講成「以太坊而家提前知出塊人，所以確認可以全面縮短」。

**Q4：點解以前唔確定？同 EIP-7251 有咩關係？**  
A：RANDAO 種子本身早已有至少 1 個 epoch 嘅確定前置，但 proposer 選舉仲要用當時嘅有效餘額。Epoch 內 EB 一變，下一 epoch 名單就可以同提前計嘅唔同。MaxEB 去到 2048 ETH 之後，存款、合併、獎勵更容易令 EB 越過整數閾值。7917 改為用更早凍結嘅 EB 來預先算名單，寫死喺 state。RANDAO 延遲本身冇改。

**Q5：合約或者我哋後台點樣讀到呢份名單？**  
A：共識層直接讀 beacon state 嘅 `proposer_lookahead`（現有 proposer duties 亦因而對下一 epoch 穩定）。執行層合約沒有「直接 RPC 讀 CL state」，而係用區塊入面嘅 beacon block root 加 Merkle proof 證明某一個 index 對應某個 validator。CEX 後台若只跑 EL，要另接 CL API，唔好假設 `eth_getBlockByNumber` 會告訴你未來 proposer。

**Q6：我哋自己有跑驗證者，要唔要改出金系統？**  
A：職責監控可以按呢份穩定名單睇下一 epoch，唔使擔心中途改卦。出金廣播、gas bump、確認政策唔使為 7917 改。若將來接 based preconf，要獨立風控：對手是邊個 proposer、承諾點失效、失效之後點重試；呢層失敗唔等於鏈出咗問題，亦唔等於可以提前入帳。

## 小練習（題目 + 參考答案）

**題目：** 組員聽完 Fusaka 簡報後打算：(1) 文件寫「lookahead 已鎖死下一個 proposer，出金保證喺佢嗰格入塊」，所以 pending 超時改成等下一個已知 slot，錯過就即時重播；(2) 對手提供 preconf，就對大額充值提早入可用；(3) 認為 7917 即係 FOCIL／FCR，確認政策可以全面縮短；(4) 只跑 EL 節點，以為 `eth_getBlockByNumber("pending")` 可以讀到未來 proposer。有咩問題？點改？

**參考答案：**

- **名單穩定 ≠ 必被包容。** 超時仍然按「未見／已見未入塊／已入塊等確認」分級。唔好對住下一個 slot 自動重播，以免卡 nonce 同重複廣播。
- **Preconf 唔係最終性。** 大額入帳繼續等實際區塊加既定確認深度。Lookahead 只係令「你同邊個約定」變得可驗證。
- **7917、FCR、FOCIL、ePBS 唔能混。** Fusaka 已上線嘅係 7917；FOCIL／ePBS 仍未上主網。確認政策唔好借題發揮全面放寬。
- **未來 proposer 喺 CL state，唔喺 EL 嘅 pending 塊。** 要接共識層 API 或驗證過的 lookahead 證明。生產掃描仍然以已出塊 receipt／logs 為準。
