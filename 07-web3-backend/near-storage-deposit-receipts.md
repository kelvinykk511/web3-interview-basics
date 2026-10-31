# NEAR 充提：named account、storage deposit、異步 receipts

## 人話
NEAR 三個特別位：
1. **帳戶有名**：`alice.near`、`xxx.tg`，亦有 64 位 hex 嘅 implicit account。地址校驗要支援兩種。
2. **Storage staking**：合約要為存你嘅資料收「押金」。NEP-141 token（似 ERC-20）要收款人先 `storage_deposit` 註冊，否則 `ft_transfer` 會失敗。
3. **異步執行**：一筆 transaction 會變成一串 **receipts**，可能跨幾個 block 執行。**Transaction 成功唔等於每個 receipt 成功**。

## 面試短答
- 充值：唔好只睇 tx 狀態，要睇**所有 receipt outcome**；NEP-141 以 `ft_transfer` event（NEP-297 `EVENT_JSON` log）+ receipt 成功為準；token 用**合約帳戶 ID** 白名單（例如 `usdt.tether-token.near`），唔好信 symbol。
- 等 `final` finality（Doomslug + BFT，約 2–3 秒）先入帳。
- 出金 NEP-141：先 call `storage_balance_of(receiver)`；冇就由交易所幫佢 `storage_deposit`（約 0.00125 NEAR，計入成本或手續費），再 `ft_transfer`（要附 1 yoctoNEAR）。
- 熱錢包用 access key（full / function-call key）簽，每個 key 有自己 nonce——可以開多個 key 做並發，避開單 nonce 瓶頸。

## 常見追問（連答案）
**Q1：`ft_transfer_call` 點解要特別小心？**
答：佢會先扣錢、再 call 收款合約 `ft_on_transfer`、最後 `ft_resolve_transfer` 可能**退返部分或全部**。所以實際到帳 = 最後 resolve 後嘅結果。掃鏈要等成條 receipt 鏈完成，以最終 event 計，唔好喺第一個 receipt 就入帳。

**Q2：出金 tx 顯示 SuccessValue，但用戶話冇收到？**
答：可能 tx 頂層成功，但後面某個 receipt Failure（例如收款人未註冊 storage，token 退返熱錢包）。要用 `EXPERIMENTAL_tx_status` / `tx` 攞齊 `receipts_outcome` 逐個檢查；失敗就標記出金失敗、對返熱錢包餘額，唔好當成功。

**Q3：Implicit account 收錢有咩要注意？**
答：64 hex 地址收到 NEAR 就自動存在；但 NEP-141 依然要 storage_deposit。仲有大小寫：NEAR 帳戶 ID 只准細階，校驗要拒絕大階，唔好幫用戶自動轉。

**Q4：Gas 用唔晒會點？**
答：Prepaid gas 用唔晒會退（以 refund receipt 形式），所以對帳時熱錢包 NEAR 餘額會有細額退款入賬，要識別為 gas refund，唔好當充值。

## 小練習（附答案）
**題：** 你掃到一筆 tx，`receiver_id = usdt.tether-token.near`，method `ft_transfer_call`，amount 1000 USDT 去你哋入金合約，tx 狀態成功。可唔可以即刻入 1000？

**參考答案：** 唔可以。
1. 等 finality = final。
2. 追齊 receipts：`ft_on_transfer` 返回「未用金額」，`ft_resolve_transfer` 會退返嗰部分畀發送者。
3. 以最終 `ft_transfer` event 嘅淨額入帳（可能少過 1000，甚至 0）。
4. 合約 ID 對白名單、amount 按 decimals（USDT 6 位）換算，用 tx hash + receipt id 做冪等鍵。
