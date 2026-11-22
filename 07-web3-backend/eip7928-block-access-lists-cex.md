# EIP-7928：Block-Level Access Lists（BAL）對 CEX 充值掃鏈／節點

> 銜接：呢幾日已經講過 Glamsterdam 嘅 [EIP-7732 ePBS](./eip7732-epbs-cex-inclusion.md)、[EIP-7708 ETH Transfer log](./eip7708-eth-transfer-logs-cex.md)、[EIP-8037／8038 狀態 gas](./glamsterdam-eip8037-8038-state-gas-cex.md)。ethereum.org 把 **EIP-7928 Block-Level Access Lists（BALs）** 寫成呢次升級嘅執行層 headliner：每個塊會附帶一份「呢個塊到底摸過邊啲帳戶／storage、執行後嘅結果係幾多」嘅地圖。對你嚟講，佢補嘅係 [非交易入帳](./eth-non-tx-balance-changes.md) 同 7708 都仲要自己拼湊嘅缺口，而唔係取代 receipt／log 掃鏈。

官方（以而家文件為準）：[EIP-7928](https://eips.ethereum.org/EIPS/eip-7928)、[Glamsterdam](https://ethereum.org/roadmap/glamsterdam/)。截至 **2026-11-22 仍未上主網**。ethereum.org（2026-10-16 更新）仍寫預計 Q4 2026、主網日期未確認，下一個里程碑曾寫 Hoodi 2026-10-27，但呢個 rehearsal 已因 8037／8038 兼容性問題延後。ACDE #249（2026-11-19）會議報道：大多數執行層客戶端傾向維持現有規格；Hoodi 暫定 12 月 14 日當週（或 16–17 日），最終決定交下一次 ACDC（11-26）；若規格不變，主網最早約 2027-01 中，較現實係 1 月底／2 月初。參數同日期仍可能改，上線前要再對一次規格。

## 人話（從你熟的 CEX）

你而家掃 ETH 入帳，要同時睇交易 `value`、internal call、fee recipient，仲有 EIP-4895 validator withdrawal（冇普通 tx）。EIP-7708 之後，部份 ETH 轉帳會多咗 Transfer log，但 11-18 已經講過：手續費、base-fee 燃燒、**validator withdrawal**、零金額同被 revert 嘅內層呼叫，7708 **唔會**出 log。

BAL 係另一條路：出塊嗰陣，執行層必須交出一份 **強制** 嘅區塊級 access list，記錄呢個塊實際接觸過嘅地址同 storage，以及寫入後嘅 **post-value**（執行後數值）。有咗呢份地圖，驗證者可以預先並行讀盤、並行校驗交易，亦可以喺唔重放每筆交易嘅情況下，用最終結果更新狀態（executionless state update）。呢點同 EIP-2930 嘅交易級 `accessList` 唔同：2930 係你簽名交易時可選填嘅「我預計會摸邊啲槽」；7928 係塊已經執行完、而且必須同實際執行一致嘅紀錄。規格寫明：2930 入面嘅項目 **唔會自動** 抄入 BAL，只有真正被 touch／改過嘅先會入。

結構上要記住三點：

1. 塊頭新欄位 `block_access_list_hash`，係 RLP 編碼後 BAL 嘅 `keccak256`。空 BAL 嘅 hash 係 `0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347`。BAL **唔放喺 block body**，執行層另行存放，經 Engine API 嘅 `ExecutionPayload` 傳（`ExecutionPayloadV4.blockAccessList`；`engine_newPayloadV5` 校驗、`engine_getPayloadV6` 產出）。計出嚟嘅 BAL 同提供嘅對唔上，塊就無效。
2. 每筆變更都帶 `block_access_index`：`0` 係執行前系統合約呼叫；`1…n` 係塊內第 1 至第 n 筆交易；`n+1` 係執行後系統操作（包括 EIP-4895 withdrawal、EIP-7002／7251 出隊等）。忽略 `n+1`，就會漏掉「冇用戶交易、但地址餘額已變」嘅入帳。
3. `balance_changes` 記嘅係該 index 結束後嘅 **絕對餘額**，唔係 delta。如果一筆交易入面先入後出、結束時餘額同交易前一樣，規格規定 **唔記** `balance_changes`（地址仍可能出現喺清單，但你冇辦法單靠呢欄知道中間過過數）。零金額收款同樣唔入 `balance_changes`。

對 CEX 掃鏈：fork 之後，可以用 BAL 幫手對原生 ETH 入帳（含 internal `value > 0` 嘅 CALL、coinbase、4895 withdrawal），但仍然要等 `finalized`（或你們既定最終性），亦唔好丢掉現有 receipt／7708 log／withdrawal 掃描。ERC-20 仍然以 Transfer event 為準——BAL 嘅 storage 係 raw slot，唔等於代幣金額。indexer／自建節點必須喺 fork 前支援 Amsterdam payload，唔係會跟唔到鏈。

大小限制跟塊 gas：`bal_items = storage_keys + addresses`，`bal_items <= block_gas_limit // 2000`。`ITEM_COST = 2000` 刻意低過最平嘅冷 `SLOAD`（2100），用來吸納唔佔塊 gas 嘅系統合約同 withdrawal 條目。

## 面試短答

1. **BAL 係區塊級、強制、帶 post-value 嘅摸觸地圖，唔係 2930 嗰種可選 access list。** 2930 條目唔會自動入 BAL；hash 對唔上，塊無效。
2. **入帳仍然等最終性。** BAL 只描述「呢個已出嘅塊做過咩」，唔係 mempool 承諾，更唔係 finalized。
3. **`balance_changes` 係絕對餘額，要自己減先有金額；同一筆 tx 淨額為零就可能完全睇唔到過數。** 系統操作喺 index `0`／`n+1`，4895 withdrawal 喺 `n+1`。
4. **ETH 入帳可以把 BAL 同 7708 log、既有掃 tx／internal／withdrawal 互補；ERC-20 仍靠 Transfer。** 7708 唔覆蓋 validator withdrawal，BAL 會記到收款後餘額。
5. **Fork 前升級節點／indexer。** BAL 唔在 body，要識 `ExecutionPayloadV4` 同新 Engine API；未支援就跟唔到 Glamsterdam。

## 常見追問（連答案）

**Q1：有咗 BAL，係咪可以唔再跑 evm、直接信 post-balance 入帳？**  
A：BAL 令「executionless 更新狀態」成為可能，但前提係呢個塊已經被共識接受，而且你拿到嘅 BAL 同 `block_access_list_hash` 一致。未 finalized 嘅塊一樣可以 reorg。另外淨額為零嘅過數唔入 `balance_changes`，單靠 BAL 唔等於看見所有資金路徑。CEX 入可用仍對齊最終性；BAL 係加速對帳同補漏，唔係新確認規則。

**Q2：佢同 EIP-7708 點分工？掃鏈要唔要砍掉舊路徑？**  
A：7708 為「有 value 嘅 ETH 轉帳」補 ERC-20 式 Transfer log，但明確唔包括手續費、base-fee 燃燒、validator withdrawal、零金額同被 revert 嘅內層呼叫。BAL 用 post-balance 補上 4895 withdrawal、coinbase、internal value 轉帳等。兩邊都要等最終性。上線初期保持雙路徑，對唔上先告警，唔好一 fork 就只信新欄位。

**Q3：出金交易填咗 EIP-2930 accessList，會唔會自動出現喺 BAL，或者影響廣播？**  
A：唔會自動抄入。BAL 只記實際執行時 touch 到嘅地址／slot。2930 仍然只影響該筆交易嘅 gas／預熱，同區塊級 BAL 係兩層嘢。出金流程唔使因為 7928 改簽名格式；要改嘅係節點、Engine API 同掃鏈。

## 小練習（題目 + 參考答案）

**題目：** Glamsterdam 一上線，組員打算：(1) 見到 `latest` 塊 BAL 入面，充值地址有 `balance_changes` 就即時入可用；(2) 用 BAL 取代 ERC-20 Transfer 掃描；(3) 只處理 index `1…n`，當系統操作同以前一樣唔影響客戶餘額；(4) 當 2930 accessList 係 BAL 嘅預覽，mempool 見到就當會入塊。有咩問題？點改？

**參考答案：**

- **仍然等 finalized：** `latest` 嘅 BAL 只係該塊嘅執行結果，可以 reorg。入可用條件唔因 7928 放寬。
- **餘額係絕對值：** 要用該 index 之前嘅餘額減出金額；同一筆 tx 淨額為零可能冇 `balance_changes`，唔等於冇資金經過。
- **ERC-20 繼續掃 Transfer：** BAL storage 係 slot，唔好當成 token 數量。
- **要處理 `0` 同 `n+1`：** 4895 withdrawal 喺 `n+1`，冇用戶 tx 都會令地址餘額變；熱錢包／質押收款地址漏掃就對唔到帳。
- **2930 ≠ BAL：** mempool 裡嘅 accessList 唔保證會被執行，更唔係入帳依據。節點／indexer 先升級到支援 `ExecutionPayloadV4`。
