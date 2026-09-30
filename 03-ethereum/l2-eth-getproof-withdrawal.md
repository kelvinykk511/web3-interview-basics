# L2 eth_getProof／Withdrawal Proof 入門

## 人話（從你熟的 CEX）

你已經識 L2 finality：optimistic rollup 由 L2 提返 L1 有 **challenge window**，產品上「橋接中」同「內部多鏈熱錢包直出」係兩條路。今日補一層：**證明本身係咩**——`eth_getProof` 拎到嘅 Merkle Patricia proof，點樣證明「呢個 L2 帳戶／storage 喺某個 state root 之下存在」，同官方橋 L2→L1 提款點用呢類 inclusion proof。

類比：CEX 出金要「帳本餘額 + 簽名審批」；rollup 提回 L1 要「L1 上已公布嘅 L2 state root + 你喺嗰個 root 下有權提嘅證明」。有 proof ≠ L1 已經可以花——仲要過挑戰窗／finalize 步驟。

## 面試短答

- **`eth_getProof(address, storageKeys[], blockTag)`**：回帳戶證明（nonce、balance、storageHash、codeHash）同指定 storage key 嘅 **Merkle Patricia Trie（MPT）proof**，相對該 block 嘅 **state root**。用途：輕客戶端／跨鏈橋驗證「某狀態真喺某 root 下」，唔使下載成個 state。
- **Optimistic L2→L1 withdrawal（概念）**：用戶喺 L2 發起 withdraw → 某時刻 L2 state root（或輸出根）被 **post 到 L1** → 用戶（或 relayer）提交 **inclusion proof**，證明自己嘅 withdrawal message／餘額變更包含喺該 root → 進入／等待 **challenge／dispute window** → 窗完且無成功挑戰 → L1 釋放／可 claim。
- **「proof verified」vs「funds spendable」**：證明只說明「相對於某個已提交 root，訊息係 inclusive」；optimistic 路徑下，**可花**通常要等挑戰期結束（或走官方 finalize／claim）。ZK rollup 則偏「validity proof 上 L1 後較快可提」，但仍受證明產生／L1 確認約束。
- **CEX 產品邊界**：用戶睇「已橋出」要對齊狀態機（`l2_initiated` → `root_posted` → `proven` → `challenge_elapsed` → `l1_claimable/done`）。若公司用 **內部多鏈錢包** 對用戶即時出 L1，背後係自家流動性／調撥，**唔等於**用戶嗰筆已經走完官方橋 proof 流程——客服／風控文案唔好混。

## 常見追問（連答案）

**Q：`eth_getProof` 到底證明咩？同 `eth_getStorageAt` 點分工？**  
A：`getStorageAt` 係「信呢個 RPC／全節點」直接讀一個 word；`getProof` 額外畀你 **可以獨立相對 state root 驗證** 嘅 sibling path（account + storage proofs）。橋、輕客戶端、爭議期要用後者；日常熱錢包對帳用前者（或 `balanceOf`）就夠。兩邊都要釘 `blockTag`／高度，唔好 latest 同 finalized 混用。

**Q：點解 L1 合約要驗 proof，唔係淨信 sequencer 一句「佢提咗」？**  
A：信任最小化：L1 只信 **已寫上 L1 嘅 state／output root** + 密碼學 inclusion。Sequencer 可以排序，但偷提要嘛通過錯誤 root（可被挑戰），要嘛攞唔到對應 root 下嘅合法 proof。CEX 若自建「快速橋」，信任模型就變返 **信自家託管／做市流動性**，要單獨披露同風控。

**Q：我已經喺瀏覽器／腳本 generate 到 withdrawal proof，係咪代表 L1 有錢？**  
A：**唔係。** Proof 只係材料；仲要：(1) 對應 root 已在 L1；(2) 提交／被接受；(3) optimistic 下挑戰窗走完（或協議規定嘅 finalize）；(4) 成功 `claim`／解鎖。窗內被挑戰成功可以令提款失敗。產品顯示「可提」必須跟 **協議狀態機**，唔好跟「我本地有一份 proof JSON」。

**Q：同之前講嘅 L2 finality／CEX 內部調撥點分工？**  
A：Finality 筆記講「點解要等、soft vs hard」；本篇講「等緊嗰段，鏈上用咩證明銜接 L1」。內部多鏈熱錢包：用戶秒到，公司承擔橋／再平衡風險。官方橋路徑：用戶（或代理）跟 proof + 挑戰窗，SLA 以日計。面試一句分清即可。

## 小練習

**題：** 用戶喺 OP Stack L2 申請提 ETH 返 Ethereum。客服見到「L2 交易已成功」同埋一條 explorer 鏈接顯示 withdrawal 已 initiate；值班同事話「我哋用 eth_getProof 拉到 account proof 啦，可唔可以喺產品標『L1 已到帳』同埋開放用戶喺 L1 提現額度」？邊度錯？正確狀態應點標？

**參考答案：**  
錯在把 **initiate + 本地／RPC 有 proof** 當成 **L1 可花**。正確順序概念上：L2 initiate → 等待該批次／output root **post 到 L1** → 提交 inclusion proof（proven）→ **challenge window** → claimable → L1 餘額可用。產品應標 `bridging`／`waiting_challenge` 之類，唔好標 L1 credited。若 CEX 用內部 L1 熱錢包先墊付，要另開「內部墊付／應收橋接」帳，同官方橋 finalize 對帳，且風控要假設挑戰期內仍可能出問題——墊付＝信用風險，唔係 proof 已等於結算完成。
