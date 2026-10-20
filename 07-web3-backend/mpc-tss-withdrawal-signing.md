# MPC/TSS 錢包出金簽名（對比 Safe 多簽）

> 銜接：[Safe 多簽出金流程（大額／冷錢包）](./safe-multisig-withdrawal-flow.md) 講鏈上 M-of-N；呢篇講 MPC/TSS 鏈下 threshold、鏈上似 EOA。

## 人話
私鑰由頭到尾冇完整出現過。用 DKG（分佈式密鑰生成）直接生成幾份 key share，放唔同機器／唔同公司（例如交易所 A 節點、B 節點、雲 HSM C）。出金時要 t-of-n 份一齊跑幾輪計算，合力出一個**普通 ECDSA/EdDSA 簽名**。鏈上睇落同普通 EOA 轉帳一模一樣。

CEX 類比：好似雙人覆核，但唔係兩個人各自蓋印（多簽＝鏈上見到兩個簽名），而係兩個人各持半條密碼一齊開鎖，外面只見到門開咗。

## 面試短答（30 秒）
MPC/TSS 用 DKG 分佈式生成私鑰，每方只持 share；簽名時 t-of-n 方跑多輪協議，產生一個標準簽名，鏈上同 EOA 無分別。好處：跨鏈通用（BTC/EVM/TRON/Solana）、gas 同普通轉帳一樣、門檻同成員可以鏈下 reshare 改而地址唔變。代價：安全靠協議實現同運維（協議 bug、share 備份、簽名服務可用性），而且鏈上冇審批痕跡，審計記錄要自己留。Safe 多簽相反：鏈上強制門檻、透明，但每條鏈要部署合約、gas 貴、非 EVM 唔通用。CEX 常見：熱錢包用 MPC 求速度同多鏈；金庫用 MPC + 多簽／HSM 疊加。

## 重點
- **同 Shamir 分片唔同**：Shamir 簽名前要喺一部機拼返完整私鑰，嗰刻係單點；TSS 簽名時都唔拼。
- **出金流程**：出金單 → 風控 policy engine（白名單、限額、KYT）→ 每個 co-signer **自己 decode raw tx** 再跑 policy → 多輪 MPC → 標準簽名 → 廣播。
- **唔可以盲簽 hash**：orchestrator 被黑可以餵假內容（Bybit 2025 事件就係簽名者見到嘅內容同實際簽嘅唔一樣）。co-signer 要獨立核對 to、amount、chainId、nonce、calldata（例如 ERC-20 只准 transfer selector，禁 approve／delegatecall）。
- **延遲／吞吐**：ECDSA TSS 要幾輪通訊（GG18/GG20 輪數多；CGGMP21 可預計算 presignature，在線一輪）；EdDSA 用 FROST 較簡單。熱錢包吞吐受簽名服務限制 → presignature 池、批量出金。
- **Presignature 一次性**：重用（或 failover 時兩個 instance 讀到同一個）會洩露私鑰，好似 ECDSA nonce k 重用。要原子 claim、先持久化 USED 再簽。
- **Key refresh / proactive resharing**：定期換 share，公鑰同地址唔變；離職員工或懷疑洩露嘅 share 即作廢。若可能已有 ≥t 份洩露，就要生新 key 搬資產。
- **已知漏洞**：2023 年 BitForge（Fireblocks 披露，部分 GG18/GG20 實現缺 zk 證明可被抽私鑰）、TSSHOCK。要用有完整證明嘅協議版本並審計實現。
- **可用性**：t-of-n 要留冗餘（如 2-of-3）；簽名超時唔好自動換新 nonce 重簽，同一出金單保持冪等。
- **審計**：鏈上睇唔到邊個批，要自己存：審批人、policy 版本、raw tx、各節點簽名 log。

## 常見追問（連答案）
1. **MPC 同 Shamir 有咩分別？** Shamir 用嗰陣要拼返完整 key（單點）；TSS 由生成到簽名都冇完整 key。
2. **MPC vs Safe 多簽點揀？** 要多鏈、低 gas、改成員唔搬地址 → MPC；要鏈上可驗證門檻、透明審計 → 多簽；大額金庫兩者疊加。
3. **點解 co-signer 唔可以淨係簽 hash？** 盲簽；上游被黑就可以換內容。每個節點自己 parse tx 同用獨立數據源核對出金單。
4. **Presignature 有咩坑？** 只可用一次；並發／failover 重用會漏私鑰 → 狀態機 + 原子 claim。
5. **一份 share 懷疑洩露？** reshare 作廢舊 share，地址唔變；如果 ≥t 份可能洩露就換 key 搬錢。
6. **簽名服務 down？** 2-of-3 仲可以簽；出金單排隊唔丟，同一單同一 nonce 保持冪等。

## 小練習
**題**：熱錢包係 2-of-3 MPC（交易所 A 節點、B 節點、雲端 C）。A 節點嘅 orchestrator 被入侵，攻擊者想偷熱錢包 USDT。佢做到啲咩？你點設計令佢做唔到？

**參考答案**：
- 一份 share 唔夠門檻，佢要騙 B 或 C 一齊簽：例如 to＝攻擊者地址，或者 to＝USDT 合約但 calldata 係 approve(攻擊者, max)（金額欄睇落係 0）。
- 防法：B、C 各自 decode raw tx，同**獨立**出金 DB 核對收款人、金額、chainId、nonce（唔信 A 傳過嚟嘅元數據）；ERC-20 只准 transfer selector；限額 + 速率限制 + KYT；出金單由出金系統用自己 key 簽名，co-signer 驗簽先簽。
- 事後：reshare 作廢 A 嘅 share，查 log。
