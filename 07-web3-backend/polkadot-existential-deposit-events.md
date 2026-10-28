# Polkadot（Substrate 系）充值出金：Existential Deposit + 用 events 掃鏈

> CEX 橋樑：EVM 你習慣「餘額 0 嘅地址照存在」、「睇 tx 就知轉咗幾多」。Substrate 系（Polkadot、Kusama 等）兩樣都唔成立。

## 人話
- **Existential Deposit（ED，最低存在餘額）**：帳戶餘額跌穿 ED，條帳戶會被「收割」（reaped），剩低嘅零頭**直接冇咗**（燒/除掉），nonce 都歸零。Polkadot 主網 ED 係 1 DOT（以官方當前參數為準，每條鏈唔同）。
- 轉帳少過 ED 去一個**新地址**會失敗——因為對方帳戶開唔到。
- 一筆 extrinsic（≈ tx）入面可以包 `utility.batch`、`proxy.proxy`、`multisig.as_multi`，真正嘅轉帳藏喺入面。所以**入帳要睇 events**（`balances.Transfer`），唔好只 parse 最外層 call。

## 面試短答
1. 充值：訂閱 **finalized head**（GRANDPA 最終性），逐 block 讀 events，認 `balances.Transfer { from, to, amount }`，同埋同一 extrinsic 有 `system.ExtrinsicSuccess`；`ExtrinsicFailed` 唔入帳。
2. 冪等鍵：`(block_hash/number, event_index)` 或 `(block, extrinsic_index, event_index)`。
3. 出金：用 `transfer_keep_alive`（保證熱錢包唔會跌穿 ED 被收割），熱錢包永遠預留 ED + 手續費。
4. 出金到新地址：金額必須 ≥ ED，前端/風控先擋，唔好等鏈上失敗。
5. 地址：SS58 格式，有網絡前綴（Polkadot 0、Kusama 2、通用 42），同一把公鑰喺唔同網絡顯示唔同地址 → 校驗時 decode 出前綴，唔啱網絡就拒。

## 常見追問（連答案）
**Q1：用戶充 0.5 DOT 去一個全新充值地址，會點？**
A：鏈上轉帳本身會失敗（對方帳戶達唔到 ED 開唔到），用戶錢冇轉出、只蝕手續費。所以充值頁要寫清「最低充值 ≥ ED」，同時最低充值額通常設得高過 ED 好多（覆蓋歸集成本）。

**Q2：歸集（sweep）充值地址時要注意乜？**
A：想清空就用 `transfer_all`（keep_alive=false）或 `transfer_allow_death`，讓帳戶被收割，唔會留低零頭；如果用 keep_alive 會永遠留 ED 喺每個充值地址，幾萬個地址 × 1 DOT 係一大筆死錢。

**Q3：點解唔可以只睇 extrinsic 嘅 call data 判斷入帳？**
A：（1）batch/proxy/multisig 會嵌套；（2）`utility.batch` 部分失敗時，前面嘅成功、後面嘅唔執行（`batch_all` 就全部 rollback）；（3）call data 寫幾多 ≠ 實際轉幾多。events 係執行結果，唯一真相。

**Q4：finalized 同 best block 揀邊個？**
A：入帳用 finalized（通常十幾秒到一分鐘內），best block 可以被重組。可以用 best block 做「到帳中」顯示，但唔加可用餘額。

**Q5：運維上仲有咩坑？**
A：runtime upgrade 會改 metadata/事件結構 → 解碼要跟 runtime version 動態載入 metadata（用 polkadot.js / subxt / Sidecar），唔好硬編 event index。另外 Polkadot 生態把餘額功能遷去 Asset Hub 嘅計劃要跟官方公告，接入節點/鏈端點可能要轉。

## 小練習（題目 + 參考答案）
**題：** 掃鏈服務只 parse 每個 block 入面 `balances.transfer_keep_alive` 嘅 extrinsic。有用戶話佢用多簽錢包充咗 100 DOT 但冇入帳。你點排查同修？

**參考答案：**
1. 用戶 tx 係 `multisig.as_multi` → 內層先係 `balances.transfer_keep_alive`，外層 parse 唔到 → 漏單。
2. 拎佢 block hash 讀 events：見到 `balances.Transfer { to: 我哋充值地址, amount: 100 DOT }` + `multisig.MultisigExecuted { result: Ok }` → 確認到帳，手動補入帳（冪等鍵 block+event_index 防重複）。
3. 根治：改為**以 events 為準**掃 finalized block，對所有 `balances.Transfer` 中 `to ∈ 我哋地址集` 入帳，同時校驗所屬 extrinsic 成功、多簽/批量內層執行結果 Ok。
4. 跑一次歷史回補，對比鏈上餘額 vs 帳本，搵其他漏單。
