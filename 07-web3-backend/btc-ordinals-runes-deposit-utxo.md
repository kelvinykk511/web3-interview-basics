# BTC Ordinals／Runes：充值地址收到「帶資產嘅 UTXO」點處理

## 人話
BTC 本身冇「代幣合約」。Ordinals 係用一套「編號規則」話某粒 sat（聰）上面刻咗嘢（inscription，例如圖片／BRC-20 指令）；Runes 係將代幣餘額寫喺交易 OP_RETURN 入面，跟住某啲 UTXO 走。
**對 Bitcoin Core 嚟講，呢啲 UTXO 只係普通 546 sat 左右嘅細額 UTXO。** 交易所錢包如果唔識，coin selection 隨手揀咗佢嚟付手續費或者歸集，個 NFT／代幣就直接燒咗或者送咗俾礦工／收款人。

## CEX 角度（對比帳戶模型）
- ETH 上 ERC-721 喺合約度，你唔 call 合約就唔會郁到。
- BTC 上「資產」= 某條 UTXO（甚至係 UTXO 入面第幾粒 sat）。**你用錯 UTXO = 你轉走咗資產。** 所以風險喺「出金／歸集」，唔係喺「充值」。

## 面試短答
> 交易所如果唔支援 Ordinals/Runes，充值地址仍然會收到帶 inscription 或 rune 嘅 UTXO。風險係錢包 coin selection 當佢普通 BTC 用咗，導致用戶資產被燒或者送錯人。做法係：充值掃描時用 ord/indexer 標記呢啲 UTXO，入 UTXO 表加 `asset_tag`，coin selection 同歸集一律排除；唔支援就唔入帳但凍結保留，走人工找回流程。如果要支援，就要按 sat 級別追蹤（Ordinals）或 rune 餘額（Runes），出金用專門構造交易：inscription UTXO 放第一個 input 對應第一個 output，padding 足夠，手續費用另一條純 BTC UTXO 付。

## 常見追問（連答案）
**Q1：點樣知道一條 UTXO 帶咗 inscription？Bitcoin Core RPC 睇唔到喎。**
A：要自己跑 `ord` indexer（或可信第三方 API，但要對帳），佢追蹤每粒 sat 由 coinbase 開始嘅流向（first-in-first-out 分配）。充值掃描入庫時 join indexer 結果，打 `has_inscription / rune_balance` 標記。indexer 落後時，新 UTXO 先標 `unknown`，**unknown 一律唔准 coin selection**，等 indexer 追上先放行——寧願慢都唔好燒資產。

**Q2：歸集（sweep）時點保護？**
A：sweep job 嘅 UTXO 查詢加條件 `asset_tag = 'NONE' AND indexer_checked = true`。另外即使係 546 sat 呢類 dust 級 UTXO 本身唔抵歸集（手續費 > 面值），都唔應該掃。重點：UTXO 表要有「可花費狀態」欄位（spendable / frozen / reserved），唔係見到就用。

**Q3：用戶將 Ordinals NFT 充錯入交易所（交易所唔支援），點算？**
A：同「充錯幣」流程類似：唔入帳、UTXO 凍結；用戶提交工單 + 證明（txid、inscription id）；人工審批後構造專門交易，把嗰條 UTXO 原封送去用戶指定 Taproot 地址（Ordinals 錢包通常用 bc1p）；手續費從另一條 UTXO 付，可收找回手續費。要注意 output 順序同金額，唔好令 sat 落入手續費。

**Q4：Runes 同 Ordinals 有咩唔同？對後台影響？**
A：Ordinals 係「某粒 sat 刻字」，非同質，追蹤到 sat 位置；Runes 係同質代幣，餘額記喺 UTXO 上，由 OP_RETURN 裏面嘅 runestone 指令決定轉去邊個 output，**冇寫清楚嘅餘額預設去第一個非 OP_RETURN output，寫錯指令（cenotaph）會燒晒**。支援 Runes 充值 = 要 parse runestone + 用 indexer 計餘額；入帳以 indexer 結果為準，同樣要確認數同 reorg 回滾。

**Q5：BRC-20 呢？**
A：BRC-20 係用 inscription 寫 JSON（deploy/mint/transfer）嘅「鏈下約定」，餘額由 indexer 按規則計，**唔同 indexer 可能計出唔同結果**。交易所支援嘅話要揀定一個參考實現、自己跑、同第三方對帳；風險明顯高過 ERC-20。

## 小練習（題目 + 參考答案）
**題：** 你哋 BTC 熱錢包某日出金，手續費估 3,000 sat，coin selection 揀中一條 546 sat + 一條 0.01 BTC 嘅 UTXO。事後用戶投訴：「我上星期充錯咗一個 Ordinals 入你哋地址，而家個 inscription 唔見咗。」點樣排查同防止再發生？

**參考答案：**
1. 排查：用 txid 查 indexer，確認嗰條 546 sat UTXO 嘅 inscription 去咗邊（通常落入第一個 output＝出金用戶，或者被當手續費俾咗礦工）。如果落入其他用戶出金地址，只能聯絡追討，鏈上唔可逆。
2. 根因：UTXO 表冇 asset 標記，coin selection 只睇金額。
3. 修正：充值掃描接 ord indexer，新 UTXO 預設 `indexer_checked=false`；coin selection/sweep 只用 `asset_tag=NONE && checked=true`；546 sat 左右嘅細額 UTXO 預設凍結。
4. 補救：建立充錯 Ordinals 找回工單流程，用專門交易送返；對帳報表加「凍結 UTXO 清單」。
