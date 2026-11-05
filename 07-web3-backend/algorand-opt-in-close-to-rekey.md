# Algorand：ASA Opt-in、Min Balance、Close-to 入金漏數、Rekey

## 人話
Algorand 每個帳戶要保持**最低餘額**（基本 0.1 ALGO，每 opt-in 一個資產再加 0.1 ALGO）。想收某個代幣（ASA），要先 **opt-in**（自己轉 0 個嗰隻資產畀自己）。
最易出事係兩個特殊欄位：
- **close-remainder-to**：一筆 payment 除咗轉 `amount`，仲可以「清戶」——將剩餘全部餘額轉去另一個地址。如果你掃鏈只睇 `amount`，就會漏數。
- **rekey**：帳戶可以將「簽名權」交畀另一條 key（`auth-addr`），地址唔變，但控制權已經唔係原本私鑰。

## 面試短答
- 冇 fork：Algorand 出塊即 final（instant finality），唔使等確認數；用 Indexer（或 algod + 自己 index）掃。
- 入金金額 = `amount` + **`closing-amount`**（close-to 收到嘅部分）；ASA 一樣有 asset close-to。仲要包埋 **inner transactions**（App 合約內部發出嘅轉帳），似 EVM internal tx。
- 出金 ASA 前，查目標地址有冇 opt-in；冇就會失敗（`asset ... missing from account`），要擋喺前面、提示用戶先 opt-in。
- 交易有 `firstValid` / `lastValid`（最多約 1000 rounds 窗口）→ 過期就唔會上鏈；txid 由內容決定，可以用同一筆簽好嘅交易重發，或用 `lease` 欄位防同一時段重複。
- 熱錢包／充值地址要監控 `auth-addr`：如果被 rekey 咗，原私鑰簽唔到，要即刻告警（可能係被盜／內部誤操作）。

## 常見追問（連答案）
1. **點解出金 ALGO 有時會報 below min balance？**
   答：出金後熱錢包餘額低過 min balance（0.1 + 0.1 × 已 opt-in 資產數）。熱錢包可用餘額要扣走 min balance，唔可以當全部可用。用戶出金去新地址都要至少 0.1 ALGO 先開到戶，金額太細會失敗。
2. **用戶出金時帶 close-to 會點？**
   答：交易所自己砌出金交易，永遠唔設 close-remainder-to／asset-close-to（設咗就係清空熱錢包！）。簽名服務要做欄位白名單：遇到 close-to、rekey-to 非空直接拒簽。
3. **掃到 rekey 交易點判斷危險？**
   答：任何 `rekey-to` 非空、而 sender 係我方地址 → 最高級別告警、暫停該錢包出金。充值地址被 rekey 意味私鑰外泄或被釣魚簽名。
4. **ASA 有 clawback / freeze address 係咩？**
   答：發行方可以設 clawback（強制收回）同 freeze（凍結）權限。上幣前要查；有 clawback 嘅資產，鏈上餘額隨時可以被發行方拎走，對帳要預咗呢種「非用戶操作」嘅減少。

## 小練習（題目 + 參考答案）
**題目**：對帳發現某充值地址鏈上收到 50 ALGO，但系統只入帳 0。查交易發現：`amount = 0`，`close-remainder-to = 你哋充值地址`。點解漏？點改？

**參考答案**：
- 用戶係用「清戶」方式轉錢：主轉帳金額係 0，真正嘅 50 ALGO 走 `closing-amount` 去咗你地址。掃鏈程式只讀 `amount` 同 `receiver`，冇處理 close-to 收款方 → 漏數。
- 修正：入帳邏輯改為「我方地址作為 receiver 收 `amount`」+「我方地址作為 close-to 收 `closing-amount`」+ inner txns，都要計；唯一鍵用 `(txid, 角色/inner 路徑)`，避免同一 tx 重複入帳。
- 兜底：每日用帳戶餘額差異對帳（期末 − 期初 = 入帳 − 出帳 − 手續費），差異自動開單，就算解析漏咗新欄位都會被發現。
