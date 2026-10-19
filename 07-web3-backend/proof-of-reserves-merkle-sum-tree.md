# 儲備證明 Proof of Reserves（Merkle sum tree）

## 人話
交易所要證明兩樣嘢：**負債**（我欠用戶幾多，即帳本總餘額）同 **資產**（鏈上我真係控制幾多），然後資產 ≥ 負債。負債用 Merkle sum tree 畀每個用戶自己驗「我有被計入、金額啱」；資產用「地址清單 + 控制權證明 + 同一區塊高度讀餘額」。

## 負債側：Merkle sum tree
- 快照時間點，每個用戶一個 leaf：`hash(salt_u || uid_hash || balance)`，同時帶 `sum = balance`（每個幣種一棵樹或者 leaf 帶多幣種向量）。
- 每個中間節點：`hash(left.hash || right.hash || left.sum + right.sum)`，sum = 兩邊之和；root 帶 **總負債**。
- 點解唔用普通 Merkle tree：普通樹只證「包含」，唔證「總和」——交易所可以喺中間節點偷偷減數。sum tree 將金額綁入 hash，任何改動都會令 root 對唔上。
- 用戶拿 inclusion proof（兄弟節點 hash + sum），自己由 leaf 計返上 root，對公佈嘅 root hash 同總額。
- **負餘額漏洞**：如果某 leaf 係負數，可以抵消總負債而 root 照樣成立。用戶只見到兄弟節點嘅 sum，睇唔到入面有冇負 leaf → 用 zk 證明（zk-SNARK / zk-STARK PoR）證明「每個 leaf ≥ 0、總和正確」，又唔洩露個人資料。
- 隱私：leaf 唔放明文 uid，用 per-user salt；proof 只畀本人下載。

## 資產側
- 公佈地址清單；每個地址證明控制權：簽一段包含交易所名 + 日期/快照高度嘅 message（防止兩間交易所 claim 同一個錢包），或者由該地址轉一筆指定金額。
- 所有鏈上餘額以**同一快照時間對應嘅區塊高度**讀（archive node 讀歷史餘額）。

## 後端實作重點（CEX 視角）
- 帳本快照要一致：DB consistent snapshot（同一事務時間點 / cut-off 高度），唔可以邊掃邊變。
- 在途資金要對齊：已扣帳未上鏈嘅出金（帳本負債已減、鏈上資產仍喺）；已上鏈未 credit 嘅充值（資產多咗、負債未加）→ 列 reconciliation 項目解釋差額。
- 槓桿/借貸用戶：負 equity 用戶點計要講清楚（唔可以當負數 leaf 抵消）。

## PoR 證唔到乜（面試加分）
- 表外負債、其他債權人、資產有冇被抵押/借出、私鑰有冇被盜、快照前後借錢粉飾（window dressing）。
- 資產用自家發嘅幣（例如 FTT）計價會虛高 → 只計主流資產、按幣種逐個比 ratio。
- 緩解：多次/隨機時間快照、持續 PoR、第三方審計、鏈上餘額歷史可查快照前後資金流。

## 面試短答
「PoR 分負債同資產。負債用 Merkle sum tree：每個節點 hash 綁住子節點金額之和，root 就係總負債，用戶用 inclusion proof 自己驗；要防負數 leaf 抵消，所以用 zk 證明每個 leaf 非負。資產側公佈地址並簽 message 證明控制權，同一區塊高度讀餘額。後端要做一致快照同在途出入金對帳。佢證唔到表外負債同快照前借錢，所以要持續 PoR 加審計。」

## 常見追問（連答案）
1. **點解要 sum tree？** 普通 Merkle 只證包含，交易所可以改中間節點總額；sum tree 將金額入 hash，改任何數 root 都變。
2. **負餘額點處理？** 用戶只見兄弟節點 sum，睇唔到入面負 leaf；用 zk range proof 證明每個 leaf ≥ 0 同總和正確。
3. **快照前借錢點防？** 單次快照防唔到；用隨機/高頻快照、鏈上可查快照前後大額流入流出、第三方審計。
4. **出金在途點算？** 帳本已扣、鏈上未出：資產 > 負債嘅差額要逐筆列明（pending withdrawal 清單），唔可以無解釋。
5. **資產簽名 message 點解要寫交易所名同日期？** 防止同一錢包畀另一間交易所 claim，或者舊簽名被重用。

## 小練習（附答案）
**題目**：4 個用戶 USDT：A=100、B=50、C=30、D=20。畫 sum tree，寫 A 嘅 proof 同驗證步驟；鏈上 USDT 資產合共 190，比率幾多？
**答案**：
- Leaf：hA(100)、hB(50)、hC(30)、hD(20)。
- 第一層：N_AB = hash(hA‖hB‖150)，sum 150；N_CD = hash(hC‖hD‖50)，sum 50。
- Root = hash(N_AB‖N_CD‖200)，總負債 200。
- A 嘅 proof：[(hB, 50), (N_CD, 50)]。A 用自己 salt 計 hA → 同 hB 合計 N_AB（100+50=150）→ 同 N_CD 合計 root（150+50=200）→ 對公佈 root hash 同總額 200，兼檢查兄弟 sum ≥ 0。
- 但 A 驗唔到 N_CD 入面係咪 C=80、D=-30（總和一樣係 50）→ 所以要 zk 證明 leaf 非負。
- 比率 190/200 = 95% < 100%，資不抵債，要追查（或者在途出金解釋差額）。
