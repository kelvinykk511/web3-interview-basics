# Lightning Network 充值 / 出金（CEX 角度）

## 人話
Lightning（閃電網絡）係 BTC 嘅「鏈下快速通道」：交易所同其他節點開 **channel（通道）**，喺通道入面互相改餘額，唔使每筆都上鏈。結果係秒級到帳、手續費好平，但係**模型同 on-chain BTC 完全唔同**：
- 冇「地址 + 確認數」，而係 **invoice（發票，BOLT11）**：入面有金額、`payment_hash`、到期時間。
- 付款一成功（收款方拿到 preimage）就**即時 final**，冇 reorg、冇確認數。
- 交易所要管「通道流動性」：你通道入面嘅錢分 **outbound（我可以付出去）** 同 **inbound（我可以收入嚟）**。

CEX 類比：on-chain 似銀行電匯（要等清算），Lightning 似你同幾間大銀行開咗「預存額度」嘅即時轉帳通道，額度用完就要補。

## 面試短答
> Lightning 充值：每次用戶要充值就由我哋節點生成一張帶金額、帶到期時間嘅 invoice，以 `payment_hash` 做訂單唯一鍵綁定用戶；監聽節點 invoice 狀態，`SETTLED` 先入帳，冪等鍵就係 `payment_hash`，唔使等確認數。
> 出金：用戶提交 invoice，後端 decode 校驗網絡、金額、到期時間、`payment_hash` 未付過，先凍結餘額再設 fee limit 發付款。關鍵係付款可能長時間 `IN_FLIGHT`，只有節點明確回 `FAILED` 先可以解凍退回，`SUCCEEDED` 要記錄 preimage 做付款證明。另外要監控通道 inbound/outbound 流動性，用 submarine swap 或開關通道喺 on-chain 同 Lightning 之間調撥。

## 充值流程
1. 用戶要充 0.01 BTC → 後端 call 節點（LND `AddInvoice`）生成 invoice，存 `payment_hash → user_id, amount, expiry`。
2. 用戶錢包付款。
3. 監聽 invoice 訂閱（LND `SubscribeInvoices`）或輪詢，狀態 `SETTLED` → 以 `payment_hash` 冪等入帳。
4. 唔好接受 **amountless invoice**（零金額發票）做充值，否則以 `amt_paid` 為準入帳，同 UI 期望唔一致；真要支持就一定用實收金額。
5. 過期未付嘅 invoice 標記 expired；但仍要處理「過期前最後一刻 settle」嘅競態 → 一切以節點最終狀態為準。

## 出金流程（重點係狀態機）
1. Decode invoice：前綴 `lnbc`（主網）/`lntb`（testnet）/`lnbcrt`（regtest）唔啱拒絕；金額要等於用戶申請金額（或 amountless 時由我哋填）；剩餘有效期太短（例如 < 幾分鐘）拒絕。
2. 查 `payment_hash` 喺本地出金表冇出現過（防同一張 invoice 提兩次）。
3. 凍結餘額 → 發 `SendPaymentV2`，設 `fee_limit`（例如金額 0.5%–1% 上限）同 `timeout`。
4. 狀態：
   - `SUCCEEDED`：記錄 preimage（付款證明）、實際路由費 → 扣帳完成。
   - `FAILED`（no route / insufficient balance / incorrect details）→ 解凍退回。
   - `IN_FLIGHT`：HTLC 卡喺路由中間，可能要等到 CLTV timeout（可以係幾個鐘到幾日）。**唔可以當失敗退款**，否則之後成功就雙付。
5. 重啟時用 `TrackPaymentV2(payment_hash)` 對返所有未完結出金。

## 流動性 / 對帳
- 收得多 → inbound 用完，再收唔到；付得多 → outbound 用完。
- 調撥方法：開新通道、submarine swap（例如 Loop Out：Lightning 錢換 on-chain）、circular rebalance。
- 對帳：節點總資產 = on-chain 錢包餘額 + 所有通道 local balance（扣 reserve）+ pending/force-close 中嘅資金。
- Force close 嘅錢要等 `to_self_delay` timelock 先返到 on-chain 錢包 → 對帳要有「鎖定中」一欄。
- 單筆金額上限要細（受通道容量同路由限制），大額仍然行 on-chain。

## 常見追問（連答案）
**Q1：點解 Lightning 充值唔使等確認數？**
A：付款成功代表 HTLC 已經用 preimage 喺我哋通道內結算，資金已經係我哋通道嘅 local balance，冇 reorg 概念。風險轉移到「通道被對手 force close + 欺詐」，由節點/watchtower 處理，唔係入帳邏輯處理。

**Q2：出金 `IN_FLIGHT` 卡咗兩日，客服話用戶催，點算？**
A：唔可以手動退款。可以話畀用戶知處理中；技術上等 HTLC timeout 由節點出最終狀態。如果最後 `FAILED` 先解凍；`SUCCEEDED` 就完成。要有告警（IN_FLIGHT 超過 N 分鐘）同人工儀表板，但最終狀態只認節點。

**Q3：同一張 invoice 用戶提交兩次出金點防？**
A：本地以 `payment_hash` 做唯一索引（DB unique key），第二次直接拒。節點層面 LND 亦會拒絕重付已成功嘅 hash，但唔可以只靠節點。

**Q4：Lightning 熱錢包同 on-chain 熱錢包有咩唔同？**
A：Lightning 節點私鑰必須 **一直在線**（要簽通道更新），冇辦法冷存；所以 Lightning 只放細額度，設獨立限額，定期將多出嘅 outbound 用 swap 搬返 on-chain 再歸集去冷錢包。

**Q5：用戶畀 Lightning Address（user@domain）而唔係 invoice 點處理？**
A：Lightning Address 基於 LNURL-pay：後端去 `https://domain/.well-known/lnurlp/user` 攞 callback，再請求指定金額嘅 invoice，之後照樣 decode 校驗（金額、description hash）再付。要防 SSRF 同 timeout。

## 小練習（題目 + 參考答案）
**題目：** 用戶申請 Lightning 出金 50,000 sats，你發付款後節點進程重啟，DB 狀態仍係 `SENDING`。你點處理？

**參考答案：**
1. 重啟後掃所有 `SENDING` 記錄，用 `payment_hash` call `TrackPaymentV2` 查節點真實狀態。
2. `SUCCEEDED` → 記 preimage、路由費，標記完成扣帳。
3. `FAILED` → 解凍退回。
4. `IN_FLIGHT` → 保持凍結，繼續追蹤。
5. 節點**完全冇呢個 hash 嘅記錄**（即係根本未發出去）→ 先確認 invoice 未過期先可以重發；過期就標失敗退款。重發前一定要確認節點冇記錄，避免雙付。
重點：**狀態以節點為準、payment_hash 冪等、IN_FLIGHT 絕不退款**。
