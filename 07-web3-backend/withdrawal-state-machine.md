# 提現狀態機（鏈下 ↔ 鏈上）

## 人話

提現唔係「扣餘額就完」：要過風控、凍結、進 signer、上鏈、等確認、失敗回滾。每個狀態都要可對帳。

## 面試短答（常見狀態）

`requested → risk_approved → frozen → broadcasting → pending_on_chain → confirmations → success`  
失敗分支：`rejected`／`broadcast_failed`／`dropped`／`reorg_rollback`（視設計）→ 解凍或人工。

要點：
- 扣款／凍結要同 `withdrawId` 冪等
- 上鏈後以 `txHash` 關聯；確認數達標先 success
- 廣播成功但久不確認：監控 + 加速／告警，唔好重複扣款

## 常見追問＋答案

Q：用戶連點兩次提現？  
A：業務冪等鍵（`clientRequestId`／提現單號）+ 狀態機；第二下返回同一單或拒絕，唔開兩次凍結。
