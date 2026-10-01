# L2 getProof Deep-dive：Prove Path／Output Root

> 銜接入門：[L2 eth_getProof／Withdrawal Proof 入門](./l2-eth-getproof-withdrawal.md)（MPT inclusion、有 proof ≠ L1 可花）。本篇再拆：**prove path 點驗**、**output root 喺 L1 代表咩**、prove vs finalize／claim、CEX 狀態機欄位。

## 人話（從你熟的 CEX）

入門篇講過：rollup 提回 L1 要「L1 上已公布嘅 L2 root + 你喺嗰個 root 下嘅 inclusion proof」。今日補「證明點砌、點對」。

想像 CEX 出金審批：你唔係把成本帳本寄畀審批員，而係畀一份 **「由根雜湊一路往下嘅路徑憑證」**——審批員只信總帳根（root），用你交嘅 sibling 雜湊重算，睇係咪落到你嗰條 withdrawal／帳戶狀態。鏈上就係 **Merkle Patricia Trie（MPT）prove path**：account trie（地址 → 帳戶 RLP）同 storage trie（slot → value）各一條 sibling 路徑，對住某個 **state root** 驗。

OP-Stack 風格再包一層：L2 唔係直接把「全世界 state root」當提款錨點，而係週期性把 **output root**（由 state root、withdrawal storage root、block hash 等組成嘅承諾）**post 到 L1**。用戶（或 relayer）之後對住呢個已上鏈嘅 output，提交 prove；挑戰窗過完先 finalize／claim。有 `eth_getProof` JSON ≠ L1 合約已 accept prove ≠ 錢已 claimable。

## 面試短答

- **Prove path（概念）**：`eth_getProof` 回 `accountProof[]`／`storageProof[].proof`——由 leaf 到 root 嘅 **sibling hashes**。驗證方用已知 **state root**（或協議定義嘅 output 裡嘅 state／message root）重算；對得上就證明「呢個帳戶／storage 字喺該 root 下」。唔使下載成個 trie。
- **Account vs storage**：先證「地址 A 嘅帳戶存在且 `storageHash=S`」，再證「喺 trie root=`S` 下，slot K 嘅值係 V」。Withdrawal message／橋合約 storage 多數落喺後者。
- **Output root（OP-Stack 風格）**：proposer 把某 L2 高度對應嘅 **output root** 寫上 L1。Output 承諾通常綁住 L2 state root、withdrawals 相關 root、block hash 等——L1 合約之後只信「已接受／未被挑戰成功嘅 output」，唔信 sequencer 口頭一句。
- **Prove vs finalize／claim**：`prove`＝把 inclusion 交畀 L1、掛上某 output／proposal；之後進入／繼續 **challenge window**；窗完且有效 → `finalize`／`claim` 先真正喺 L1 釋放。三個時間點產品要分開標。
- **CEX 狀態機欄位（建議）**：`l2_initiated` → `output_proposed`（root／index 已在 L1）→ `proven`（L1 已記你條 withdrawal）→ `challenge_elapsed` → `l1_claimable` → `l1_claimed`／`done`。內部墊付另開 `internal_advance`／應收，唔好同 `l1_claimed` 混。

## 常見追問（連答案）

**Q：Prove path 驗成功，係咪等於 L1 合約已經「認」呢筆提款？**  
A：**未必然。** 本地／腳本對住某 root 重算成功，只證明材料密碼學上對；仲要：(1) 該 output／proposal 已在 L1 且係協議接受嘅那條；(2) 成功呼叫 prove（或等效）寫入橋合約；(3) 過挑戰期；(4) claim。CEX 監控應跟 **L1 橋合約事件／狀態**，唔好只跟「我哋 worker 生成咗 proof JSON」。

**Q：Output root 同 state root 係咪同一個嘢？**  
A：**唔係同一個名詞。** State root 係某個 L2 block 執行後世界狀態嘅 MPT 根；output root 係協議定義嘅 **打包承諾**（裡面會引用 state root／withdrawal 相關根等），專門畀 L1 錨定同爭議。面試講：「L1 錨嘅係 output／proposal；getProof 對嘅係 trie 相對某 state／message root——兩者透過協議公式接埋。」

**Q：點解要分 prove 同 finalize，唔係一次 tx 搞掂？**  
A：Optimistic 要留 **爭議窗口**：prove 之後、放錢之前，挑戰者可以針對錯誤 output／欺詐證明攻擊。合成一步會令偷提同糾正搵同一個原子窗口。ZK 路線可以「validity 上鏈後較快提」，但仍然有證明產生同 L1 確認延遲——產品狀態機唔好照抄「一筆 receipt 就 done」。

**Q：CEX 若用官方橋代用戶 prove／claim，風控要盯咩欄位？**  
A：至少：L2 initiate tx／message nonce、對應 **output index／root**、L1 prove tx、challenge 截止時間、claim tx、同內部是否已墊付。對帳用「output 已 propose 但未 proven」「proven 但窗未完」「窗完未 claim」分桶老化；墊付倉要假設窗內仍可能失敗，唔好一 proven 就當結算完成。

## 小練習

**題：** 值班見到：L2 withdrawal 已 initiate、`eth_getProof` 對某 `blockTag` 拉到 storage proof 且本地 verifier 對住「最新 L2 state root」通過；但 L1 上該批次嘅 output 仍未 propose。產品經理想標 `proven` 並開始倒數 7 日挑戰期。可唔可以？應標邊個狀態？

**參考答案：**  
**唔可以。** 本地對住「最新 L2 state root」通過，只說明材料可能有用；OP-Stack 風格提款要對住 **已 post 到 L1 嘅 output**（同協議要求嘅那組根）去 prove。Output 未 propose → 最多標 `l2_initiated`／`waiting_output`；output 上 L1 後先有資格提交 prove → `proven`；其後先倒挑戰鐘 → `challenge_elapsed` → claimable。把「本地 getProof OK」當成「挑戰期已開始」，會令 SLA、客服同墊付風控全面錯位。
