# Solana SPL 出金：收款人冇 ATA 點算

## 人話
Solana 上 token 唔係放喺錢包地址本身，而係放喺「錢包 × mint」派生出嚟嘅 token account（ATA）。用戶俾你嘅出金地址如果未有呢個 ATA，你要幫佢開戶，開戶要押 rent（165 bytes 嘅 rent-exempt 約 0.00203928 SOL）。呢筆錢由 payer（即 CEX）出，但押金歸新 account；之後用戶 close 戶口，錢退去用戶指定嘅地方，唔係 CEX。所以對 CEX 嚟講係**成本**，唔係押金。

## 出金 tx 點砌
- 同一筆 tx：`createAssociatedTokenAccountIdempotent`（已存在就 no-op）+ `transferChecked`（帶 mint 同 decimals，program 會驗證）。
- 用 Idempotent 版：避免用戶同時開咗 ATA 或者你自己重試時，普通 create 遇到「已存在」令成筆 tx fail。

## 地址校驗
- 出金前 `getAccountInfo(目的地址)`：
  - 冇 data / System Program 擁有 → 正常 wallet 地址，按 owner derive ATA。
  - 係 Token Program 擁有嘅 token account → 用戶填咗 token account 地址。唔好當 owner 再 derive（如果係 on-curve 嘅非 ATA token account，會開出一個冇人有 key 控制嘅 ATA，錢卡死）。要麼拒絕並提示，要麼按政策直接轉入並驗證其 mint 一致。
- derive ATA 時預設唔准 off-curve owner（PDA），避免誤把 PDA 當用戶錢包。

## Token-2022
- ATA 地址同 token program id 有關：Token-2022 mint 要用 Token-2022 program id derive。
- extension（例如 transfer fee）會令收款少過發出；有 extension 嘅 account size 更大、rent 更多。
- mint 白名單要記錄屬於邊個 program。

## 成本 / 風控
- 新 ATA rent（約 0.002 SOL）遠大過 base fee（每個簽名 5,000 lamports = 0.000005 SOL）。
- 對首次開戶收「開戶費」或計入出金費；設最低提幣額。
- 防濫用：攻擊者用大量新地址細額提幣燒你 rent → 最低額 + 開戶費 + 頻率限制。
- 確認：用目的 ATA 餘額變化 + tx 結果確認到帳。

## 面試短答
Solana 上 SPL token 存喺 owner × mint 派生出嚟嘅 ATA。出金時如果收款人未有 ATA，要喺同一筆 tx 先 createAssociatedTokenAccountIdempotent 再 transferChecked，CEX 做 payer 墊約 0.002 SOL rent，呢筆唔會退返俾我哋，所以要計入出金費或最低提幣額。仲要校驗用戶填嘅係 wallet 定 token account，Token-2022 要用對 program id derive ATA。

## 常見追問（連答案）
- **Q：點解唔直接 transfer 去用戶錢包地址？** A：SPL transfer 嘅目的地必須係 token account；錢包地址本身唔係，冇 ATA 就 fail。
- **Q：點解用 Idempotent 版？** A：已存在時普通 create 會令成筆 tx fail；idempotent 存在就跳過，重試安全。
- **Q：用戶之後 close ATA，rent 去邊？** A：退去 close 時指定嘅 destination（由 owner 決定），CEX 收唔返。
- **Q：transfer 定 transferChecked？** A：transferChecked，帶 mint + decimals 俾 program 驗證，防 mint 錯或 decimals 搞錯金額；Token-2022 有 transfer fee 要處理收款少過發出。

## 小練習（附答案）
一日 10,000 筆 USDC 出金，30% 收款人冇 ATA。rent 0.00203928 SOL，每筆 1 個簽名、base fee 5,000 lamports。每日 SOL 成本（未計 priority fee）？

**答案**：開戶 3,000 × 0.00203928 = 6.11784 SOL；tx fee 10,000 × 0.000005 = 0.05 SOL；合共約 6.17 SOL，rent 佔約 99%。所以開戶費要另外收。
