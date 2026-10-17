# TRON Stake 2.0 — 交易所點樣養一個「能量池」

> 銜接：[TRON 地址激活 + 資源租賃成本](./tron-address-activation-energy-rental.md)、[Solana ATA 充值 + TRON Energy](./solana-ata-deposit-tron-energy.md)。

## 人話
- 喺 TRON 轉 TRC-20 USDT 要燒 energy，冇 energy 就燒 TRX。交易所唔想每次歸集都燒錢，所以用一個「資源主帳戶」質押 TRX 換 energy，再借俾充值地址用。
- Stake 2.0 將「質押」同「代理」拆開：
  1. FreezeBalanceV2：質押 TRX，揀 ENERGY 或 BANDWIDTH，同時得到投票權（TRON Power）。
  2. DelegateResource：將資源借俾另一個地址（例如充值地址），可以揀加 lock（鎖定期內收唔返）。
  3. UnDelegateResource：收返。
  4. UnfreezeBalanceV2：解質押，要等 14 日先可以 WithdrawExpireUnfreeze 攞返 TRX；等緊期間可以 CancelAllUnfreezeV2 取消（重新質押）。
- Energy 唔係固定匯率：你攞到嘅 energy ≈ 你質押嘅 TRX ÷ 全網為 energy 質押嘅 TRX × 全網每日 energy 總量。全網質押多咗，同樣 TRX 攞到少咗。用完喺 24 小時內慢慢回復。

## CEX 對照
好似喺雲廠商買咗預留容量，平時分俾各服務用；臨時爆量就短租（能量租賃市場）；租唔到就燒 TRX（最貴嘅按量付費）。

## 面試短答
「我哋用一個資源主帳戶 FreezeBalanceV2 質押 TRX 換 energy，歸集前用 DelegateResource 將 energy 代理到充值地址，歸集完 UnDelegate 收返，循環用。質押量按每日歸集筆數 × 每筆 energy 估，尖峰靠租賃或者允許燒 TRX 兜底，fee_limit 封頂。解質押有 14 日等待，所以質押嘅 TRX 喺資金規劃要當非流動資產，唔可以當熱錢包可用餘額。」

## 常見追問（連答案）
1. 問：Stake 1.0 同 2.0 最大分別？
   答：1.0 質押同代理係一步，凍結 3 日；2.0 拆開，質押之後可以隨時代理／收回俾唔同地址，啱交易所輪流借俾大量充值地址；解質押改為 14 日等待期，期間可以取消。
2. 問：代理咗出去嘅資源可唔可以直接解質押？
   答：唔得，要先 UnDelegate 收返嗰部分先可以 Unfreeze。如果代理時加咗 lock，鎖定期未過連收都收唔返，所以交易所內部輪轉代理通常唔加 lock 或者 lock 好短（lock 多數用喺賣能量俾第三方，保障買家）。
3. 問：點估要質押幾多？
   答：每日歸集筆數 × 每筆 USDT transfer 嘅 energy（目標地址未持有 USDT 時大約雙倍）÷「每 TRX 每日可得 energy」（從鏈上即時計，會浮動），再加 buffer。監控使用率，長期唔夠加質押，短期尖峰就租。
4. 問：資源帳戶揸住大量 TRX，點保護？
   答：用 TRON 帳戶權限（owner / active permission）：owner 用冷錢包或多簽；俾熱端服務一條 active permission，只允許 DelegateResource / UnDelegateResource 呢類合約類型，唔允許 TransferContract。熱端 key 漏咗都轉唔走 TRX。
5. 問：質押得到嘅投票權有咩用？
   答：可以投 SR（超級代表）攞投票獎勵，呢部分係公司收入，唔係用戶資產。要分清交易所自己嘅 TRX 質押同用戶參加嘅質押理財產品，後者要喺負債記錄同儲備證明度對應返用戶。

## 小練習
題：每日要歸集 2 萬筆 TRC-20 USDT，月尾有一日去到 6 萬筆，唔想長期為尖峰質押咁多 TRX，點設計？
參考答案：
- 底倉：按平日 2 萬筆加約 20% buffer 質押，energy 每日回復，輪轉代理俾當批要歸集嘅地址。
- 尖峰：(a) 唔急嘅小額地址延後歸集（歸集唔影響用戶到帳，可以排隊）；(b) 能量租賃市場短租；(c) 最後兜底允許燒 TRX，fee_limit 封頂，只做大額地址（燒費要細過金額某個比例先歸集）。
- 監控：energy 剩餘、燒 TRX 金額、每筆歸集成本告警；成本超標自動減慢歸集。
- 資金：質押嘅 TRX 標記為非流動，解質押要 14 日，唔可以臨急當出金資金。
