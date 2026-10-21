# MPC 簽名服務 HA + presign 池

> 銜接：[MPC/TSS 錢包出金簽名（對比 Safe 多簽）](./mpc-tss-withdrawal-signing.md) 講協議同 co-signer；呢篇講高可用、presign 池同重試。

## 人話

MPC 簽名要幾個節點一齊計，一個慢／掛就簽唔到；**presign** 係預先整好嘅「半成品」，簽嗰陣通訊輪數少好多，但**每份只可以用一次**。

## 面試短答

用 **t-of-n**（例如 2-of-3）換可用性；節點分唔同雲／AZ／團隊，share 放 HSM／TEE；presign 池按 key 分池、有水位預熱、嚴格一次性；**coordinator** 只負責調度，掛咗只影響可用性唔影響安全，可多實例。

## 重點

- presign 狀態機 **`available → reserved → consumed`**，先落庫再用；crash 後 **`reserved` 一律當廢棄唔回收**。
- 協議中途有節點冇回應 → 整輪 **abort** 換組重試（**identifiable abort** 可揪出搞事節點）。
- 同一出金單重試要簽**同一 payload**（同 nonce），但用**新 presign**；ECDSA 簽名唔同所以 EVM **txhash 都唔同**，同 nonce 只會上一張，監控要追蹤全部 hash。
- **reshare／key refresh** 後舊 presign 全部作廢。
- 監控：簽名 **P99**、池剩餘量、**abort 率**、節點健康。

## 常見追問（連答案）

1. **presign 重用會點？**  
   等同 ECDSA nonce 重用：兩個唔同 message 用同一 nonce 就可以推算私鑰（PS3 事件同一原理），所以**絕對一次性**。

2. **重簽會唔會雙出？**  
   EVM 同一 nonce 只會成功一張，所以唔會雙出，但要記錄**所有 txhash** 去對帳；危險係重試時生成**新 nonce** 嘅新 tx。

3. **2-of-3 有一個節點長期掛？**  
   仲簽到但冇冗餘，盡快 **reshare** 換新節點（地址唔變），期間**調低限額**。

## 小練習（連參考答案）

**題**：出金高峰 presign 池用晒，簽名延遲由 200ms 升到 3s，上游 timeout 重試令請求翻倍，點處理？

**參考答案**：

- **短期**：降級做在線完整簽名但**限流**；重試帶**冪等 key** 合併成同一請求，**exponential backoff**，防重試風暴；按優先級排隊。
- **長期**：按出金預測預熱池、低水位告警、背景並發補充、按 key 做容量規劃。
- **絕不**為快而重用 presign。
