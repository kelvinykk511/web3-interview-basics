# EIP-7251：MaxEB 2048 同 validator consolidation（CEX 質押後台）

> 銜接：質押生命週期、0x02 可 compound 至 2048 ETH、4895 sweep，見 [eth-staking-product-backend.md](./eth-staking-product-backend.md)。執行層觸發 exit／部分提款見 [eip7002-el-triggerable-withdrawals.md](./eip7002-el-triggerable-withdrawals.md)。今日專注 **Pectra 之後點樣把大量 32 ETH validator 合併、同 0x01／0x02 記帳差異**。

系統合約（mainnet）：`0x0000BBdDc7CE488642fb579F8B00f3a590007251`。同 7002 一樣係 withdrawal 地址在 EL 排隊、CL 先校驗，EL receipt 成功 ≠ 已合併。

## 人話（從你熟的 CEX）

以前每個 validator 好似一個「額度 32 ETH 嘅子帳戶」：獎勵超過 32 就自動 sweep 返冷錢包，要加大規模就只能再開多個帳戶、多把 BLS 操作鑰。Pectra 嘅 **EIP-7251** 容許一種 **0x02 compounding** 帳戶，有效餘額上限去到 **2048 ETH**，獎勵可以留喺同一個 validator 複利，亦可以用 consolidation 把舊嘅 32 ETH validator **併入**一個 0x02 目標。

好似把公司幾百個戶口歸集到少數總戶口：簽名機器同監控都會少，但 **一把鑰出事嘅殺傷力大好多**。仲有一條同 7002 唔一樣嘅風控：consolidation 嘅目標，**協議唔要求**同來源用同一個 withdrawal 地址——來源 WC 一旦被盜，攻擊者可以把本金併去自己嘅 0x02 validator，錢唔會再 sweep 返你原來嗰個冷錢包。

## 面試短答

1. **兩種憑證**：**0x01** 仍然係 max EB **32 ETH**，超出部分定期 4895 自動 skim；**0x02** max EB **2048 ETH**，超過 2048 先 skim，之下嘅獎勵留喺 validator 複利。激活下限仍係 **32 ETH**（`MIN_ACTIVATION_BALANCE`）。新開可以存款時直接用 0x02；已有 0x01 要先做 **self-consolidation**（calldata 裡 source pubkey＝target pubkey）先轉類型，唔能夠默默喺合併途中改 creds。
2. **呼叫方式**：由 **來源 validator 嘅 withdrawal 地址** 向系統合約轉帳（`msg.value` 蓋過當前 fee）。Calldata **96 bytes**＝source pubkey（48）＋ target pubkey（48）；`source_address` 唔寫喺 calldata，合約用 `msg.sender`。空 calldata 可 `eth_call` 讀 fee；最低 **1 wei**，目標每塊 **1** 筆、每塊最多 dequeue **2** 筆，超出就指數升，**多付唔退**。隊列硬上限 **262,144**，超出會被 CL 丟棄且費用不退。
3. **兩步流程**：目標必須 **已經係 0x02**。正確做法多數係：先把目標（或其中一個）self-switch 成 0x02，再把其他 source 併入。CL 會核對：來源有執行層憑證且地址對得上、兩邊都 active、未開始 exit、來源已激活滿約 **256 epochs（≈27 小時）**、來源冇 pending partial withdrawal。對唔上就 **靜默跳過**，區塊仍然有效。
4. **錢幾時先過去**：CL 接受後，source 進入 exit（用 **以餘額加權** 嘅 consolidation churn，大批合併要排期），期間 source **仍要當值賺獎勵**；到 `withdrawable_epoch`（exit ＋ **256 epochs ≈ 27 小時**）先把餘額撥入 target，**唔經 4895、唔打去冷錢包**。若喺 pending 期間 source 被 **slash**，consolidation **取消**，餘額按退出路徑處理，唔好當一定轉到 target。
5. **風控同帳本**：目標 pubkey 必須 **白名單**，並且核對目標 WC 係本所控制嘅 0x02 地址（Lido 類協議亦係咁做）。協議 **唔保證** source／target 地址相同。狀態機：`EL_REQUEST_SENT → CL_ACCEPTED（pending consolidation）→ BALANCE_MERGED`（或 `CANCELLED_SLASHED`）。合併完成前，ETH 仍記在 source；完成後只係 stake 從 source 行轉去 target，**唔好當贖回可用**。Source 操作鑰要留到 withdrawable 之後先退役。單鑰實際承載建議自設遠低於 2048 嘅上限，因為罰則同 effective balance 成正比。
6. **加倉**：已有 0x02 可以用同一 pubkey 再 deposit 做 **top-up**（金額可以低過 32 ETH），唔好當成新開一個 validator。產品 NAV 要讀 beacon 餘額應計；0x02 可能好耐都冇 4895 sweep，唔等於冇收益。0x02 轉返 0x01 並唔係對等操作，規劃時當單向。

## 常見追問（連答案）

**Q1：EL tx `status=1` 係咪已經合併？可以唔可以刪 source 嘅簽名鑰？**  
A：唔係。系統合約只係排隊，CL 先校驗，失敗會靜默跳過。就算已受理，都要等 exit 隊列＋約 27 小時先撥帳。Source 在 exit_epoch 前仍要出籤，而且到 withdrawable 前仍然可被 slash。鑰要等合併成功（或確定取消）先退役。

**Q2：點解話 WC 被盜比 7002 更危險？**  
A：7002 全退／部分提款最終仍 sweep 去 **原來個 WC 地址**（若係多簽／timelock 合約，攻擊者仲要過你嘅提款策略）。Consolidation 可以把餘額撥去 **另一個 0x02 validator**，目標 WC 可以唔同；一旦完成，本金跟新目標走。所以目標必須白名單，WC 要當最高權限冷端。

**Q3：合併期間獎勵同對帳點計？**  
A：Source 在 exit 隊列仍然賺；進入 withdrawable 前嗰段約 27 小時就唔再當值。撥帳係 CL 餘額轉移，scanner 唔會見到一筆入冷錢包嘅 withdrawal。帳本只係把「已質押」從 source_id 轉去 target_id，切忌先減後加做成雙計或當可用。

**Q4：Fee 同排隊同 7002 有咩差別？**  
A：機制相似（先讀 fee、設上限、多付唔退；任何人都可以高價塞隊列）。但 consolidation 每塊目標只有 **1** 筆、最多處理 **2** 筆，大批從成千上萬個 32 ETH validator 遷移會好慢，要按 churn 排期，唔好假設一晚完成。Fee 高位要熔斷，唔好無上限連轟。

## 小練習（題目 + 參考答案）

**題目：** 運維要把 2,000 個 0x01（各 32 ETH）併入一個新建嘅 0x02 目標，計劃承載約 64,000 ETH。Job 見到系統合約 tx 成功就當已歸集，把 source 簽名鑰刪走，並把「在途 ETH」改為冷錢包可用。同時腳本嘅 target pubkey 係運維即場填嘅，冇校驗 WC。有咩問題？正確做法？

**參考答案：**

- **上限同風險**：單一 0x02 最多 **2048 ETH** 有效餘額，64,000 ETH 要分成多個目標（並自設單鑰上限，降低雙簽殺傷力），唔能全塞入一個。
- **狀態機錯**：EL 成功只係請求已上鏈。要等 CL pending consolidation 完成先改帳；期間刪 source 鑰可能導致 source 缺簽甚至 slashing，合併亦可能被取消。錢唔會打入冷錢包，唔可以改成可用餘額。
- **目標未白名單**：協議允許目標 WC 同來源唔同，填錯或被調包等於把本金轉走。必須預先登記本所控制、已確認 0x02 憑證嘅 target 集合，合約／腳本拒絕未知 pubkey。
- **正確流程**：先確認目標已係 0x02（必要時先 self-switch）；分批提交、監控 fee 同 churn；以 Beacon 狀態為準（source exiting／pending、之後 target 餘額增加、source 餘額清掉）；鑰管理同帳本過帳綁在 `BALANCE_MERGED`。回歸要覆蓋「EL 成功但 CL 跳過」同「pending 期間 source 被 slash」。
