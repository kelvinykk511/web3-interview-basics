# Solana / TRON vs EVM 帳戶模型（CEX 充值掃描 + 出金視角）

> 銜接：[Native vs ERC-20 充值](./native-eth-vs-erc20-deposit.md)、[熱錢包 nonce 分配](./hot-wallet-nonce-allocation.md)、[充值最終性](./deposit-reorg-confirmation-depth.md)。今日講：接新鏈時，**邊啲 EVM 假設會爆**。

## 人話（CEX 類比）

- **EVM**：token 餘額記喺 token 合約入面一張 `mapping(address => balance)`，好似「每隻幣一個大 Excel，一行一個用戶」。充值就係掃 `Transfer` event，`to` 係唔係我哋地址。
- **TRON**：本質同 EVM 好似（TVM 兼容，TRC-20 一樣有 `Transfer` event），但手續費唔係 gas，而係 **energy / bandwidth 資源**；**冇 account nonce**；新地址要「激活」。
- **Solana**：「乜都係 account」。程式（合約）本身無狀態，狀態放喺獨立 account。**每個用戶 × 每隻 token 有一個獨立 token account**（通常係 ATA）。即係唔係「大 Excel 一行」，而係「每人每幣開一個獨立子帳戶」。

## 一張表記住

| 維度 | EVM | TRON | Solana |
|---|---|---|---|
| 地址 | secp256k1，`0x`+20 bytes | 同條私鑰同 20 bytes，前綴 `0x41`，base58 `T...` | ed25519 公鑰 32 bytes，base58 |
| token 餘額放邊 | token 合約 storage | 同 EVM | 獨立 token account（owner=錢包、mint=幣） |
| 充值偵測 | `Transfer` log（contract+to） | 同 EVM，但要睇 receipt `SUCCESS` | tx meta `pre/postTokenBalances`（按 owner+mint）或盯 ATA |
| 防重放 | account nonce（要排隊） | **無 nonce**：ref block + expiration | **無 nonce**：recent blockhash（~60–90s 過期）或 durable nonce |
| 手續費 | sender 付 gas（native） | energy（合約執行）+ bandwidth（tx 大細），冇就燒 TRX | 每簽名 base fee + priority fee；**fee payer 可以係第三方** |
| 收款前提 | 任何地址都收得 | 新地址要激活（首次收 TRX 時由 sender 多付費） | token account 要先存在，且要 rent-exempt 押金 |
| 最終性 | `safe` / `finalized` | solidified block（~19 塊，約 1 分鐘） | commitment：processed / confirmed / finalized |

## 面試短答

接 TRON 可以大量複用 EVM scanner（Transfer log、合約白名單、冪等鍵），但要改三樣：用 solidified block 入帳、檢查 receipt 係 `SUCCESS`（失敗交易都會上鏈）、出金唔使 nonce 分配器而係處理 expiration。
接 Solana 要重寫 scanner：token 唔係記喺錢包地址，而係記喺 (wallet, mint) 對應嘅 token account；入帳睇 `postTokenBalances - preTokenBalances`，按 `owner ∈ 我哋地址` 同 `mint ∈ 白名單` 過濾，用 `finalized` commitment 入帳。

## 常見追問（連答案）

**Q1：TRON 同 EVM 用同一條私鑰，係咪可以共用地址同 scanner？**
A：私鑰→20 bytes 相同，只係編碼唔同（`0x41` 前綴 + base58check）。所以錯鏈充值（ETH 地址打咗 TRON）理論上可以用同私鑰找回。但 scanner 唔可以直接共用：RPC 唔同（TRON HTTP API / gRPC）、入帳要睇 solidified、交易失敗狀態欄位唔同。可以共用嘅係**帳本層**：冪等鍵 `(chain, txid, eventIndex)`、狀態機、合約白名單。

**Q2：TRON / Solana 無 nonce，對出金有咩影響？**
A：好處：唔使 EVM 嗰套「自管 nonce 分配、填洞、同 nonce 替換」，可以並發簽名。代價：交易有**過期時間**。出金卡住時，正確做法係：等到確定已過期（TRON 過 expiration；Solana 當前 block height > `lastValidBlockHeight`）**而且鏈上查唔到**，先用新 ref block / blockhash 重簽。未過期就重簽 = 可能雙出金。冪等：內部 withdrawal id 綁住「當前有效 txid」，換 txid 要記版本。

**Q3：TRON 失敗交易都上鏈，scanner 點中伏？**
A：TRC-20 call 可能 `OUT_OF_ENERGY` / `REVERT`，交易照樣喺塊入面，仲收咗費。如果只睇「有冇呢筆 tx + input 解碼出 transfer(to, amount)」就會假入帳。要用 receipt `result == SUCCESS` 加 `Transfer` event，同 EVM 睇 `status == 1` 一樣。

**Q4：Solana 點解唔可以好似 EVM 咁「filter logs by to address」？**
A：token 轉帳指令嘅目標係 **token account（ATA）**，唔係錢包地址；錢包地址未必出現喺交易 account keys 入面。所以要麼盯住我哋每個 ATA（`getSignaturesForAddress(ata)`），要麼掃全塊然後用 tx meta 嘅 `postTokenBalances[].owner` 對我哋地址。

**Q5：Solana `confirmed` 夠唔夠入帳？**
A：`confirmed` 係超級多數投票確認，實務上好少回滾，但大額 / 可提現建議用 `finalized`（約 32 slots，十幾秒）。同 EVM 一樣：`processed` 只寫 `seen`。

## 小練習（題目後即附參考答案）

**題：** 你個 EVM 充值系統要擴展支持 TRON USDT 同 Solana USDC。列出最少要改嘅 4 樣嘢。

**參考答案：**
1. **地址生成 / 驗證**：TRON 用 base58check（`T...`），Solana 用 ed25519 新 key（唔可以由 EVM key 推出）；前端地址格式校驗按鏈分開，防錯鏈。
2. **掃描邏輯**：TRON 沿用 Transfer log 但加 `SUCCESS` 檢查 + 合約白名單（USDT 正牌合約）；Solana 改為按 `owner+mint` 讀 token balance 差額。
3. **最終性**：TRON 用 solidified block；Solana 用 `finalized` commitment，各自配置。
4. **出金 / 歸集手續費**：TRON 要 energy/bandwidth（質押或代理），Solana 由熱錢包做 fee payer；兩條鏈都冇 nonce，改用 expiration / blockhash 過期後先重簽。
（加分：帳本層、冪等鍵格式 `(chain, txid, index)`、狀態機完全唔使改——呢個就係抽象做得好嘅證明。）
