# Solana durable nonce — 出金審批太耐，交易過期點算？

> 銜接：[Solana 出金：blockhash 過期重簽 + priority fee](./solana-withdrawal-blockhash-priority-fee.md)、[Solana 充值：commitment 層級、reorg 與 CEX 入帳](./solana-deposit-commitment-reorg.md)。

## 人話
- 普通 Solana 交易要帶 recent blockhash，大約 150 個 block（一分鐘左右）就過期。出金要多簽／冷錢包／HSM 審批，審完已經過期。
- Durable nonce：開一個 nonce account（System Program 管、放少量 SOL 做 rent 押金、有一個 authority）。交易用 nonce account 入面存嘅 nonce 值代替 recent blockhash，而且第一條指令必須係 AdvanceNonceAccount（nonce authority 簽）。交易一上鏈，nonce 值就更新，同一張交易唔可以再上。
- 即係：唔會過期，但只可以用一次，好似 EVM nonce；不過一個 nonce account 同一時間只承載一張待上鏈交易。

## CEX 對照
好似出金單嘅一次性審批 token：簽好冇時限，用完即作廢；要撤單就主動作廢個 token。

## 面試短答
「Solana 交易靠 recent blockhash 防重放，大約一分鐘就過期，唔啱要人工或多簽審批嘅出金。我哋用 durable nonce：每張出金單綁一個 nonce account，第一條指令係 AdvanceNonceAccount，用 nonce 值代替 blockhash，簽完幾耐先廣播都得；上鏈後 nonce 推進，天然防雙出。要撤銷就自己 advance nonce，令已簽交易作廢。並發出金用 nonce account 池。」

## 常見追問（連答案）
1. 問：durable nonce 有咩新風險？
   答：已簽交易唔會過期，外洩咗任何人隨時可以廣播。所以已簽 tx 要當敏感資料（加密存放、最少接觸）；出金單取消或者超時就主動 advance nonce 作廢，唔好留住長期有效嘅支票。
2. 問：點做並發？
   答：一個 nonce account 同一時間只可以俾一張待上鏈交易用。維護 nonce account 池，DB 記錄 nonce account → 出金單嘅租用狀態，加鎖分配，上鏈或作廢後釋放。
3. 問：廣播後唔知有冇上鏈，可唔可以重發？
   答：可以重發同一張已簽交易，同一個 nonce 值只會成功一次，唔會雙出。確認結果查 signature 狀態；如果 nonce 值已經變咗但 signature 唔係成功，就要分清係執行失敗（失敗都會推進 nonce 兼扣手續費）定係被主動作廢，再決定要唔要用新 nonce 重新簽（重簽即係重新審批）。
4. 問：priority fee 點處理？
   答：compute unit price 喺簽名嗰刻已經定死，審完網絡塞車都加唔到，要加就要重簽重審。所以預先設偏寬鬆嘅 CU price，或者審批流程容許「同金額同地址、只改 fee」嘅快速重簽通道。
5. 問：同 Squads 呢類鏈上多簽有咩分別？
   答：Squads 係鏈上 program，提案、投票、執行都喺鏈上，透明但要多筆交易；durable nonce 配合離線／MPC／HSM 簽名，簽名喺鏈下完成，最後一筆上鏈。交易所常見 MPC + durable nonce。

## 小練習
題：一張 500 SOL 出金單，風控審批平均 30 分鐘，有時審到一半用戶取消，畫個流程。
參考答案：
1. 建單：從 nonce pool 分配一個 nonce account（DB 鎖住），讀當前 nonce 值。
2. 構造交易：第一條 AdvanceNonceAccount，之後 transfer；simulate 定 compute unit limit，設 CU price。
3. 審批 + 簽名：MPC／多簽完成，已簽 tx 加密存放，狀態 SIGNED。
4. 廣播：發送，未確認可以重發同一張；大額等 finalized 先標 SUCCESS。
5. 用戶取消：未廣播就發一筆 AdvanceNonceAccount 作廢，確認 nonce 值已變先標 CANCELLED 同釋放 nonce account；已廣播就唔可以當取消，要等結果。
6. 對帳：nonce 值變咗就查 signature，成功入帳；失敗就退回或重新建單（重新審批）。
