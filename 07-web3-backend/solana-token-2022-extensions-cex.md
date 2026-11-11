# Solana Token-2022 擴展：CEX 上架／充提要注意咩

> 銜接：SPL 出金開 ATA、Token-2022 要用對 program id 派生，見 [solana-spl-withdrawal-ata-creation.md](./solana-spl-withdrawal-ata-creation.md)、[solana-ata-deposit-tron-energy.md](./solana-ata-deposit-tron-energy.md)。EVM 側類似嘅「非標準代幣」見 [non-standard-erc20-token-listing.md](./non-standard-erc20-token-listing.md)。今日專注 **Token-2022 extensions 點樣打破「轉 100 就入 100、只有 owner 可以動幣」**。

Program id（資產 key 仍然係 `(chain, mint)`，另外要記 program）：經典 SPL Token `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`，Token-2022 `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`。ATA 種子包含 token program id，用錯 program 會派生出另一個地址，錢可以轉錯或者卡死。

## 人話（從你熟的 CEX）

經典 SPL 好似「規矩嘅 ERC-20」：轉帳金額對等、只有 token account owner 可以轉出。Token-2022 唔改呢個帳戶模型，但 mint 可以掛擴展，效果好似 fee-on-transfer、可升級 hook、發行方後門、加密餘額一齊出現。

你可以當上架一隻「可能有隱藏條款」嘅 Solana 幣。只記 mint、當佢係普通 USDC 嚟掃餘額，會出現：用戶話轉咗 100、你只收到 99（轉帳費）；或者 ATA 一開就凍結，幣入到但 sweep 唔到；或者發行方嘅 permanent delegate 直接從你熱錢包扣走；又或者餘額係加密嘅，RPC `amount` 根本唔係你以為嘅明文。呢啲都唔係 reorg，係 mint 規則。

## 面試短答

上架前用 mint account 枚舉 extensions，逐項定政策，唔好只認 program id。CEX 最少要處理：

1. **Transfer Fee**：每筆按 basis points（可另有 max fee）扣費，收款方 `amount` 只增加淨額，扣咗嘅費先 withhold 喺帳戶入面，由 `withdraw_withheld_authority` 收走。**入帳記淨額**（目的 ATA 可用餘額差額），唔好按來源減少量入帳。出金用 `transferChecked`／fee 版指令，並向用戶展示「到帳淨額」。fee 可以由 authority 改（分 older／newer、按 epoch 切），要監控，超過上架時批准嘅上限就暫停充提。
2. **Permanent Delegate**：指定地址可以**唔使 owner 簽名**就從任何 token account 轉走或燒毀。等同一把發行方後門／凍結後沒收。托管熱錢包隨時可以被划走，多數情況 **拒上架**；若合規穩定幣必須接受，要當發行方託管風險寫入產品，並監控 delegate 地址同異常轉出。
3. **Transfer Hook**：每次轉帳要 CPI 去 hook program，可以 reject 或有副作用（allowlist、合規檢查）。Sweep／出金可能無端 fail，亦要喺 tx 帶齊 hook 要求嘅 extra accounts。hook program 若可由 authority 更換，等於可升級後門，當高風險。
4. **Default Account State = Frozen**：新 ATA 一開就凍，未 thaw 唔能轉出。充值可能入到凍結戶，scanner 見到餘額但歸集失敗。要嘛上架前拒絕呢種 mint，要嘛有解凍流程同「已入帳但不可 sweep」狀態，唔好當到帳完成。
5. **Confidential Transfer**、**Non-Transferable**、**Memo Transfer**、**Pausable**：加密餘額令普通 RPC 對帳失效，一般拒作托管現貨；不可轉帳就唔能出金；要求 memo 嘅轉入無 memo 會 fail（用戶資產仲喺自己度）；pausable 同 EVM 穩定幣 pause 一樣，暫停時出金／歸集會失敗，要監控。

Interest-bearing／Scaled UI Amount 令 **UI 金額變、raw amount 可以不變**，帳本應以 raw 為準，展示另算，唔好用 UI 當負債單位（對比 wstETH vs  rebasing stETH）。

## 常見追問（連答案）

**Q1：點解 ATA 一定要帶 Token-2022 program id？用經典 Token program 派生會點？**  
A：ATA 係 `findProgramAddress([wallet, tokenProgramId, mint], ATA_PROGRAM)`。用錯 program 會得到另一個地址。對 Token-2022 mint 轉去「經典程序派生出嚟嘅 ATA」會失敗或（更糟）開出一個唔屬於正確 program 嘅戶口。出金／充值地址派生、`getAccountInfo` 校驗，全部要以 mint 實際 owner program 為準。

**Q2：轉帳費扣喺邊？用戶充 100、你熱錢包 ATA 見到 100，但有部分係 withheld，點入帳？**  
A：可用餘額（token account 嘅 `amount`）只升淨額，例如 99；1 個單位係 withheld，owner 花唔到，最終歸 fee authority。入帳 99。如果錯誤把 `source 減少量` 或「account 總增量含 withheld」當成用戶負債，對帳同出金都會爆。歸集後亦唔好以為 withheld 係平台收入。

**Q3：Permanent delegate 同 freeze authority 有咩分別？**  
A：Freeze 係凍結帳戶，令你暫時轉唔出，但幣仲喺戶口。Permanent delegate 可以直接轉走或 burn，熱錢包可以無預警少幣。後者對 CEX 托管係資損模型，唔係「暫停提現」模型。兩者都可以並存，上架要分開評估。

**Q4：Hook 令 estimate／模擬成功但上鏈失敗，點處理？**  
A：同 EVM `estimateGas` 一樣，hook 依賴狀態（allowlist、配額、帳號狀態）。出金／sweep 要以交易結果為準，失敗唔好盲重試同一個可能被 hook 永久拒絕嘅地址；要記錄 hook program、模擬時帶齊 extra accounts，並把「hook 拒絕」從「RPC／blockhash 過期」分開告警。

**Q5：Interest-bearing 係咪好似 stETH 咁 balance 自己變？**  
A：唔完全一樣。好多實作係 raw `amount` 不變，UI 金額按利率／倍率變。如果帳本鎖 UI，會同鏈上 raw 對唔上。應記 raw，產品條款寫清楚息歸用戶定歸平台；若歸用戶，要用獨立應計科目，而唔係當無主充值。

## 小練習（題目 + 參考答案）

**題目：** 商務要上架一隻 Solana「合規穩定幣」，mint 喺 Token-2022。盡調發現：transfer fee 30 bps（authority 可改）、permanent delegate 指向發行方、default account state 係 Frozen、有 transfer hook。對方話「別的所已經上了，fee 同凍結都是合規需要」。工程建議點回？若最終只接受其中一部分，充提要改咩？

**參考答案：**

- **Permanent delegate**：默認拒絕。發行方可以隨時從托管地址划走資產，CEX 對用戶嘅負債對唔上鏈上控制權。除非法務／風險書面接受「發行方託管風險」並有保險或上限，否則唔上架。
- **Default frozen**：新充值 ATA 唔 thaw 就 sweep 唔到。要有官方 thaw 流程、超時告警，以及「已偵測到餘額但未解凍、未入帳或入帳但鎖住」嘅狀態，禁止當普通可用餘額。
- **Transfer fee**：可以做，但入帳／出金一律用淨額，報價要展示到帳量；監控 fee 變更，超過批准上限（例如 30 bps）即暫停。唔好寫死 30 bps。
- **Transfer hook**：要審計 hook program、權限可唔可以換實現、extra accounts 列表；出金／歸集模擬失敗要 fail-closed。hook 可升級 ≈ 可加後門，應同 permanent delegate 一齊升級風險。
- **回歸**：用測試 mint 走一次「轉 100 入帳 99.7、凍結戶不能 sweep、hook 拒絕時出金單失敗並解凍用戶餘額、delegate 划走觸發資損告警」。經典 SPL 假設全部要關閉。
