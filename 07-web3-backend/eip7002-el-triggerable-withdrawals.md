# EIP-7002：執行層觸發退出／部分提款（CEX 質押後台）

> 銜接：交易所自己跑 validator、32 ETH／Pectra 後 0x02 可compound 至 2048 ETH、獎勵經 EIP-4895 `block.withdrawals` 入帳，見 [eth-staking-product-backend.md](./eth-staking-product-backend.md)。用戶持有嘅 LST 記帳見 [lst-rebasing-steth-wsteth.md](./lst-rebasing-steth-wsteth.md)。今日專注 **Pectra 之後，持有 withdrawal credentials 嘅一方可以在執行層觸發 exit，唔再睇 BLS 操作鑰面色**。

系統合約（mainnet）：`0x00000961Ef480Eb55e80D19ad83579A64c007002`。同宗 Pectra 仲有 consolidation 合約 `0x0000BBdDc7CE488642fb579F8B00f3a590007251`（EIP-7251 合併 validator），機制相似但用途唔同。

## 人話（從你熟的 CEX）

質押有兩把「鑰」：日常當值簽名嘅 **BLS 操作鑰**（熱、可能喺外包 staking vendor／DVT），同埋決定錢歸邊度嘅 **withdrawal credentials**（應該係你哋冷錢包或合約，0x01／0x02 指住執行層地址）。Pectra 之前，只有操作鑰可以簽 voluntary exit——等於「錢係你嘅，但走唔走要睇值班嗰把鑰」。vendor 唔合作、鑰丢失、預簽 exit 用完，本金就卡住。

EIP-7002 把「提出離職」呢個掣交返畀真正嘅物主：由 **withdrawal 地址自己喺執行層發一筆 call**，把請求排入系統合約隊列，之後共識層再處理。好似公司戶口（冷錢包）而家可以主動辭退外包值班員，唔使對方簽字。但辭職信交咗，唔等於即刻收到離職金——仲要行原來嘅 exit 隊列、可提取延遲、再等 EIP-4895 sweep。

## 面試短答

呼叫 7002 系統合約：

1. **Calldata 56 bytes**＝48-byte validator pubkey ＋ **big-endian** `uint64 amount`。`amount = 0` 係 **full exit**；`amount > 0` 係 **部分提款，單位 gwei**（主要畀 0x02 compounding validator 減倉，通常唔能低過保持 active 嘅下限，約 32 ETH）。
2. **`msg.value` 要蓋過當前 fee**。空 calldata 可以 `eth_call` 讀 fee；最低 **1 wei**，隊列超過目標（每塊 target **2** 筆、每塊最多 dequeue **16** 筆）就類似 1559 **指數升**。多付**唔退**。合約應先讀 fee、設上限再 call；EOA 難精準，模擬之後略為多付。
3. **執行層合約唔驗證你係唔係物主**，只收錢排隊。**共識層**先核對 `msg.sender` 是否該 validator 嘅 0x01／0x02 執行層地址。對唔上、已經 exiting、pubkey 無效等，**唔會令區塊無效，請求被靜默跳過**（同類比：EL deposit）。所以 **EL receipt `status=1` ≠ 已開始退出**。
4. **0x00 BLS withdrawal credentials 用唔到 7002**，要先做一次憑證轉換變成 0x01／0x02。
5. 7002 **唔跳過**共識層 exit 隊列、withdrawable 延遲（約 256 epochs ≈ 27 小時）同 4895 sweep。狀態機應係：`EL_REQUEST_SENT → CL_ACCEPTED（beacon 狀態 exiting／pending partial）→ WITHDRAWABLE → SWEPT`。用戶贖回唔可以在 EL tx 確認就當 ETH 已到手。
6. 營運價值：staking vendor 只持 BLS、交易所持 withdrawal 地址時，**唔再依賴 vendor 簽 exit 或預簽**；操作鑰丢失都可以由冷端回收。相對地，withdrawal 地址而家權更大，必須當最高權限冷端（建議合約＋限額／timelock），因為攻擊者有呢把鑰就可以觸發退出，本金最終仍 sweep 去同一地址。

## 常見追問（連答案）

**Q1：點解唔能夠「tx 成功就當已退出」？**  
A：系統合約只係排隊。CL 處理時先對 credential，失敗唔回滾呢塊、亦唔一定有一條好用嘅 EL event 當最終結果。要以 Beacon API 睇 validator status／exit epoch／pending partial withdrawal。唔到位就告警重查（可能 fee／排隊成功但身份唔對，或已經喺退出中），唔好再盲目對同一 pubkey 狂發（費會升，而且多餘請求可能被跳過）。

**Q2：Fee 點估？隊列塞爆會點？**  
A：先 `staticcall` 空 calldata 取 fee，再加 `requestFeeLimit`。目標每塊 2 筆，超額 fee 指數升，任何人只要肯付費都可以把隊列塞滿做 griefing，所以要有熔斷，唔好喺 fee 高位無上限連轟幾百個 validator。每塊最多 16 筆被送入 CL，大量退出要分批，預留排隊時間。費唔退，更唔好寫死 1 wei。

**Q3：部分提款同 4895 自動 sweep 有咩分別？**  
A：0x01／0x02 本來就會把超過有效餘額上限嘅獎勵自動 sweep（0x01 常見超過 32 ETH 嘅部分）。7002 嘅 `amount > 0` 係你主動請求抽一筆指定 gwei（0x02 可以把本金減到下限附近），唔使等自動規則。錢到位仍然走 4895 withdrawals，scanner 繼續讀 `block.withdrawals`，唔好等一筆普通 tx。

**Q4：Withdrawal 地址係 Safe／合約，點 call？**  
A：必須由 **credential 地址自己做 `msg.sender`**，冇辦法用另一個 EOA「代簽」去頂替（系統合約用 `CALLER`）。Safe 要有一條模組／交易：先讀 fee、再 `call{value: fee}(pubkey || amount)`。批簽同廣播要考慮 fee 喺等待期間上升，超過 limit 就失敗重簽。

**Q5：同 EIP-7251 consolidation 點分？**  
A：7002 係 exit 或部分提款；consolidation 合約（`0x0000…7251`）係把一個 validator 併入另一個，用滿 2048 ETH 上限、減少要維護嘅 validator 數量。都係 withdrawal 地址在 EL 提交、CL 校驗、有動態 fee 同隊列。唔好把 consolidation 當成「即時到帳贖回」。

## 小練習（題目 + 參考答案）

**題目：** 你哋 ETH 質押產品由外包 vendor 跑 2,000 個 0x02 validator，withdrawal credentials 係交易所冷端合約。產品要分批贖回 10,000 ETH。同事寫咗個 job：冷合約對每個 validator 發 7002（`amount=0` 全退），看到 EL tx `status=1` 就喺帳本把對應本金從「已質押」轉去「可用」，再讓用戶提現。有咩問題？正確做法？

**參考答案：**

- **過早入帳**：EL 成功只代表請求進隊，唔代表 CL 接受，更唔代表 ETH 已 sweep 到冷錢包。提前把負債變「可用」等於未收到錢就畀用戶提走，形成缺口。
- **不該全部 `amount=0`**：只需要 10,000 ETH 時，應對 0x02 用部分提款（gwei），保留 validator 繼續工作；全退會進入 exit 隊列、停獎、還要再 deposit 才能回來，而且 2,000 筆會打爆 7002 隊列、fee 急升。
- **狀態機**：`EL_REQUEST_SENT`（記 tx hash、pubkey、amount、已付 fee）→ 輪詢 beacon，確認 pending partial／exiting → `WITHDRAWABLE` → 觀察 `block.withdrawals` 到 credential 地址先 `SWEPT`，然後先入平台質押資金科目，再按產品規則解凍用戶或從流動性 buffer 墊付。
- **Fee 同限速**：讀當前 fee + 上限；每塊最多真正進入 CL 16 筆，要排程；fee 超限暫停 job。
- **校驗**：抽樣確認 `msg.sender` 真係 credential 合約、pubkey 對得上自家 validator 名單；對「EL 成功但 CL 幾輪後仍 active 且無 pending 請求」開告警，唔好當已贖回。
- **回歸**：模擬「EL 成功但故意用錯 credential／已 exiting 被跳過」——帳本不可解凍；模擬 sweep 到帳先入帳。
