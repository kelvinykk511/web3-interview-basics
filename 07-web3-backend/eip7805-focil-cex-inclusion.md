# EIP-7805 FOCIL：包容名單同 CEX 出金審查抵抗

> 銜接：你已經識 [EIP-7732 ePBS](./eip7732-epbs-cex-inclusion.md)（邊個起 execution payload）、[出金 pending 監控](./hot-wallet-pending-monitor.md)、同 [出金 gas bump](./withdrawal-gas-bump-ops.md)。FOCIL 唔改最終性，亦唔決定交易喺塊入面排第幾；佢要解決嘅係：**公開 mempool 入面一筆合法交易，會唔會被少數 builder 系統性排除。**

官方（以 2026-11-25 文件為準）：[EIP-7805（Draft）](https://eips.ethereum.org/EIPS/eip-7805)、[ethereum.org：Hegotá](https://ethereum.org/roadmap/hegota/)（頁面更新 2026-10-16）。FOCIL **唔喺 Glamsterdam**，而係 Hegotá 共識層 headliner；同執行層 headliner [EIP-8141 Frame Transactions](https://eips.ethereum.org/EIPS/eip-8141) 一齊列為已 Scheduled for Inclusion。Hegotá 預期主網 **2027 Q2，日期未確認**，下一里程碑寫 Sepolia 2027。截至 2026-11-25 仍未上主網。ACDC #191（2026-11-12）把 FOCIL 同後量子／未來 decoupled consensus 嘅相容性問題延後決定；ACDE #249（2026-11-19）寫明，合併嘅 Hegotá 執行＋共識 devnet 會停到 ACDC 釐清 FOCIL 方向。下一次相關會議係 [ACDC #192](https://ethereum-magicians.org/t/all-core-devs-consensus-acdc-192-november-26-2026/30388)（2026-11-26 14:00 UTC），議程有 Hegotá，但喺會議出結果前，唔好當規格已凍結。

## 人話（從你熟的 CEX）

而家以太坊多數區塊唔係驗證者自己逐筆揀交易，而係少數專業 builder 出 payload（就係 ePBS／PBS 想約束嘅嗰層）。對 CEX 出金嚟講，呢個等於：**廣播成功 ≠ 一定會被打包**。如果主要 builder 對某類地址或某筆交易審查（合規名單、私人訂單流優先、甚至單純忽略低費交易），你嘅出金可以喺公開 mempool 一直待，前線只見到「已送出、未上鏈」。你而家能做嘅，係加 tip、換 RPC、等下一個 proposer，或者走私人通道——協議本身冇逼 builder 收你呢筆。

FOCIL（Fork-choice enforced Inclusion Lists）把「呢幾筆應該被收」嘅決定，由單一 builder 分散到一個委員會。規格重點如下（全部以 EIP-7805 為準）：

1. **每個 slot 抽 16 個驗證者**做 IL committee（`IL_COMMITTEE_SIZE = 16`）。
2. **Slot N、約 t=0–8 秒：** 委員按自己見到嘅公開 mempool 各組一張 inclusion list（IL），處理完本 slot 區塊（或 t=7 秒仍未收到塊就用本地 head）之後廣播。每張 IL 嘅 RLP 交易總大小上限 **8 KiB**（`MAX_BYTES_PER_INCLUSION_LIST = 8192`）。組名單策略由客戶端決定：可以隨機、按優先費、或按等待時間。
3. **View freeze 喺 slot N 嘅 t=9 秒：** 之後新收到嘅 IL 唔再納入呢一輪視圖（仍可轉發）；equivocation 證據仍會記。Builder 約喺 t=11 秒凍結自己嘅 IL 視圖，請執行層把 IL 交易補進 payload。
4. **Slot N+1 開始：** proposer 廣播滿足 IL 約束嘅區塊。t=4 秒 attester 核對：自己凍結視圖入面、非 equivocation 嘅 IL 交易，要麼已出現喺 payload，要麼補喺塊尾會因 nonce／餘額無效，或塊入面剩餘 gas 不夠。否則唔投票。執行層新狀態 `INCLUSION_LIST_UNSATISFIED` 會交返給共識層。
5. **條件式包容（conditional inclusion）：** 交易失效或塊已滿，允許唔收；FOCIL **唔規定交易排喺塊嘅邊個位置**（anywhere-in-block）。
6. **1-out-of-N：** 只要有一個誠實委員把你嘅交易寫進 IL，審查就要同時收買或令成個委員會失效。同一委員如果廣播兩張唔同 IL，會被當 equivocator，其 IL 被忽略。委員**冇額外獎勵**，機制假設利他參與。
7. **「Same-slot」係相對舊方案：** 喺 slot N 先出現嘅交易，可以成為 slot N+1 區塊嘅約束，唔使再額外推遲一個 slot（對照舊嘅 forward IL，例如 EIP-7547）。佢**唔保證**你喺送出嗰個 slot 已經入塊。

對交易所最緊要嘅邊界：

- **只覆蓋委員見到嘅交易。** 只送私人訂單流、冇進公開 mempool 嘅出金，FOCIL 幫唔到。
- **唔係最終性，亦唔係確認政策。** 入塊之後，入帳／放行仍然要等你哋自己嘅 `safe`／`finalized` 規則。FOCIL 唔阻止 reorg，亦唔保證排序，所以仍然可以夾擊。
- **唔保證下一塊必中。** 委員未必收錄、8 KiB 有限、交易可能失效、塊可能已滿。
- **而家未上線。** 唔好把出金 SLA 寫成「Hegotá 之後一定 12 秒內入塊」。升級後節點要跟新 Engine API（例如 `engine_getInclusionListV1`、`newPayload` 帶 IL、`forkchoiceUpdated` 的 payload attributes 擴展）。

同 ePBS 嘅分工：ePBS 管「payload 點樣被提出同披露」；FOCIL 管「披露出嚟嘅 payload 最少要收邊啲公開交易」。兩者互補，唔互相取代。Glamsterdam 講 ePBS 時，唔好順便假設 FOCIL 已經一齊生效。

## 面試短答

1. **FOCIL 係 Hegotá 共識層 headliner（EIP-7805），唔喺 Glamsterdam，截至 2026-11-25 未上主網。** 預期隨 Hegotá 喺 2027 Q2，日期未釘；規格同測試節奏仍待 ACDC 釐清。
2. **16 人委員會各出一張最多 8 KiB 嘅包容名單，下一 slot 嘅區塊要滿足呢啲名單，否則 attester 唔投票。** 審查唔再係單一 builder 可以單獨完成。
3. **條件式包容、唔指定排序。** 失效或 gas 不夠可以跳過；交易可以放喺塊嘅任何位置。
4. **只保護進入公開 mempool、並且被至少一個誠實委員收錄嘅交易。** 私人通道、未見嘅交易、塊滿、nonce／餘額唔啱，都不在保證範圍。
5. **唔等於加快最終性，亦唔能單靠佢放寬 CEX 入帳。** 出金仍要廣播到公開 mempool，上鏈後照自己嘅確認政策；硬分叉先生效。

## 常見追問（連答案）

**Q1：FOCIL 係咪代表出金一定喺下一個 slot 入塊？可以改短確認時間嗎？**  
A：唔可以咁理解。最快都係「slot N 被委員收錄 → 約束 slot N+1 嘅塊」，而且要交易仍然有效、塊有位。FOCIL 處理嘅係審查抵抗，唔係確認深度。大額出金／入帳繼續跟 `finalized` 或現有政策。

**Q2：同 ePBS、FCR 點分？**  
A：ePBS 係 Glamsterdam 路線，改區塊生產同 payload 披露。FCR 係共識客戶端對已出塊嘅快速確認（改 `safe` 語義），唔逼交易入塊。FOCIL 係 Hegotá 路線，用 fork choice 逼下一塊收 IL 入面嘅公開交易。三樣解決三種問題，唔好混成「以太坊加快咗所以確認可以全面縮短」。

**Q3：我哋出金只走私人 builder 通道，FOCIL 會保護嗎？**  
A：唔會，除非同一筆亦出現喺委員能見到嘅公開 mempool。想要 FOCIL 嘅審查抵抗，就要公開廣播（可以同時送私人通道追速度）。面試可以講：審查抵抗靠公開可見性；延遲靠私人通道，兩者要分開設計。

**Q4：如果 16 個委員都審查，或者有人故意出兩張唔同 IL，點算？**  
A：機制係 1-out-of-N：全部不誠實先會失效。同一委員出兩張唔同 IL 會被忽略，等於呢一票作廢，唔能用「左右各講一套」去搗亂 fork choice。委員冇協議內獎勵，所以實際效果取決於客戶端實作同誠實參與率；呢點面試要講得出，唔好吹成絕對保證。

**Q5：塊已滿、或者 IL 入面有筆交易 nonce 唔啱，整個塊會唔會被否決？**  
A：唔會單單因此否決。Attester／EL 會檢查「漏咗嘅交易若加喺塊尾是否仍然有效、同剩餘 gas 夠不夠」。無效或裝唔下就跳過。所以 FOCIL 唔係「名單上每一筆都必須出現」。

**Q6：而家要唔要改出金系統去等 FOCIL？**  
A：而家唔好把生產 SLA 綁上去。要跟 ACDC／Hegotá 範圍，預留節點同 Engine API 升級。系統層面可以先做嘅，係確保出金會進公開 mempool、監控「長時間未入塊」、同把審查延誤當成獨立事故類型，而唔好同 gas 不足、nonce 卡住混為一談。

## 小練習（題目 + 參考答案）

**題目：** 組員聽完 Hegotá 簡報後打算：(1) 文件寫「FOCIL 上線後出金保證下一個 slot 入塊」，所以 pending 超時由 3 分鐘改成 15 秒就當失敗重廣播；(2) 為咗搶速度，出金只送私人 builder，唔再進公開 mempool；(3) 認為 FOCIL 同 FCR 一齊之後，大額出金可以改跟 `safe`、唔再等 `finalized`；(4) 假設 Glamsterdam 升級當日 FOCIL 一齊生效。有咩問題？點改？

**參考答案：**

- **唔好寫「下一個 slot 必中」。** FOCIL 只約束被委員會收錄、仍然有效、且塊有空間嘅公開交易；目標係 slot N+1，唔係送出當下嗰個 slot。超時可以縮，但 15 秒一刀切重廣播，容易造成卡 nonce 同重複發送。超時要分「未見過／已被審查式忽略／已入塊等確認」。
- **私人通道唔受 FOCIL 保護。** 要審查抵抗就必須公開廣播；私人通道只作為加速，唔能取代。
- **FOCIL 唔等於 FCR，更唔等於最終性。** 大額／不可逆繼續用 `finalized`。FCR 只係確認規則，而且公共 `safe` 唔能假設已開。
- **FOCIL 唔喺 Glamsterdam。** 升級清單要分開跟；Hegotá 日期未釘，方向仍待 ACDC。而家只做監控同公開廣播，唔改生產確認政策。
