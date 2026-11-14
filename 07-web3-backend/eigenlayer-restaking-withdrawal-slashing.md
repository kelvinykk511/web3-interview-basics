# EigenLayer 再質押：提取延遲同罰沒（CEX 理財／庫存）

> 銜接：[交易所 ETH 質押後台](./eth-staking-product-backend.md)、[EIP-7002 執行層退出](./eip7002-el-triggerable-withdrawals.md)、[EIP-7251 consolidation](./eip7251-validator-consolidation.md)、[stETH rebase vs wstETH](./lst-rebasing-steth-wsteth.md)。今日唔重複 validator 點退出，而係講 **把 ETH／LST 再委託去 Operator／AVS 之後，錢幾時先真正回到托管地址、中間仲可唔可以被 slash**。

## 人話（從你熟的 CEX）

普通 ETH 質押，你已經識：用戶買「活期質押」、背後係一堆 validator，贖回要排隊，到帳靠 EIP-4895 sweep，仲有協議層罰沒。EigenLayer 再加一層：**同一筆已質押資產再借出經濟安全畀 AVS**（各種驗證服務）。用戶／協議把原生 ETH（經 EigenPod）或 LST 存入 strategy，委託俾 Operator；Operator 再把份額配置去一個或多個 Operator Set。AVS 可以按自己規則罰沒已配置嘅 Unique Stake——唔一定要鏈上可客觀證明。

對交易所後台，最容易踩嘅坑係把「已 queue 提取」當成可用庫存，或者把 LRT（例如再質押收據代幣）當成 1:1 可即時兌 ETH。正確類比係：**有鎖定期同haircut 風險嘅託管倉**，唔係現貨熱錢包。

主網安全延遲（官方 Safety Delays；治理可改，**要以合約讀數為準，唔好寫死**）：

- **提取**：`queueWithdrawals` 之後要等 `WITHDRAWAL_DELAY` = **100,800 blocks（約 14 日）**，先至唔再被 slash，先可以 `completeQueuedWithdrawal`。罰沒升級前曾經係約 7 日，而家係 14 日。
- **隊列中仍然可被罰**：文件同合約都寫明，queue 之後 deposit shares 會從可委託餘額拿走，但 **完成前仍 slashable**；`complete` 時實收可以少過排隊時嘅數量。
- **解除配置／退出 Operator Set**：`DEALLOCATION_DELAY` 同樣約 14 日。
- **罰沒落地**：`SLASH_RESOLUTION_DELAY_BLOCKS` = 50,400 blocks（約 7 日）之後先 permissionless `clearBurnOrRedistributeShares`。標準 Operator Set 會 burn（ERC-20 去死地址 `0x00…e16e4`，方便 LRT 對帳）；**可再分配**嘅 set 會打去建立時指定嘅 `redistributionRecipient`。原生 ETH 同 EIGEN 而家**不能**走再分配；原生 ETH 被 slash 後會**永久鎖在 EigenPod**，兩邊都拎唔走。
- **`undelegate`** 會把該 staker **全部** shares 排隊提取，唔係只退一部分。
- **原生 ETH** 另外要先完成共識層退出：exit queue（通常短、擁擠時理論無上限）＋固定 256 epoch（約 27 小時）＋ validator sweep（視 index，約 0–9 日）＋ EigenPod **checkpoint**，資金到齊 Pod 之後先（或重疊）走上述 14 日托管。重疊可以縮短總時間，但要預先 queue **準確金額**：估少咗 Pod 會剩；估多咗要等呢輪 escrow 完先改。

`completeQueuedWithdrawal` 嗰陣，`receiveAsTokens: false` 唔係提幣，而係把已可完成嘅提取**重新restake**返去賺獎勵。帳本唔可以見到「complete 成功」就當 ETH 入咗熱錢包。

測試網把提取延遲設成約 25 blocks（約 5 分鐘）。**絕對唔好把 testnet 常數拷去生產。** 主網請讀 `minWithdrawalDelayBlocks()`。

## 面試短答

EigenLayer 再質押係把 ETH／LST 委託俾 Operator 去支援 AVS，因而多咗 AVS 罰沒同約 14 日提取延遲。CEX 若做再質押理財或把再質押資產當庫存，狀態要分「已委託／提取排隊中／可完成／已到托管地址」；`queueWithdrawals` 只係開始計時，隊列中仍可被 slash，實收以 `completeQueuedWithdrawal` 之後托管地址實際到帳為準。原生 ETH 仲要先走信標鏈退出＋sweep＋EigenPod checkpoint。LRT 係另一層包裝，唔等於 1:1 即時 ETH。Operator／AVS 白名單係風控決策，可再分配 slashing 風險更高。

## 常見追問（連答案）

**Q1：用戶喺 App 撳「解除再質押」，鏈上 queue 成功，可唔可以即日讓他提現 ETH？**  
A：唔可以拿這筆當已到帳。主網大約要 14 日先能 complete，而且中間被 slash 可能少收。若產品要「即時贖回」，只能用自有流動性墊付，並且把墊付曝險計入 slash 同延遲；協議路徑仍要等 complete 成功、代幣入埋托管地址，先把在途對銷。唔好喺 queue 嗰刻減「已質押」同時加「可用」。

**Q2：LST 再質押同原生 ETH 再質押，提取有咩分別？**  
A：LST／ERC-20 strategy 主要就係 queue → 約 14 日 → complete 收到對應代幣（到帳量已扣罰沒）。原生 ETH 要先讓 validator 退出並 sweep 入 EigenPod，且完成 checkpoint，否則提取會失敗；之後先有（或刻意重疊）EigenLayer escrow。呢段 beacon 路徑同 EIP-7002／4895 筆記一致，只係收款地址係 EigenPod 而唔係你嘅熱錢包。

**Q3：上架 LRT（再質押流動代幣）算唔算已經處理咗 EigenLayer 風險？**  
A：未。LRT 本身係另一張 ERC-20，發行方先係 EigenLayer 嘅 staker。CEX 入帳仍然按該代幣 Transfer／實收，但計價、贖回同風控要當佢跟住底下 strategy／Operator／AVS 嘅罰沒同排隊，而唔係 ETH 現貨。唔好開「LRT 即時 1:1 兌 ETH」除非你自己備足流動性同吸收缺口。

**Q4：罰沒 burn 同 redistributable 對托管有咩分別？交易所自己做 Operator 呢？**  
A：普通 set：被 slash 嘅 ERC-20 約 7 日後 burn 掉；原生 ETH 鎖死在 Pod。Redistributable set：約 7 日後可以轉去 AVS 指定收款地址，用戶資產可能流走而唔只係銷毀，風險更高，要額外盡調。若交易所自己當 Operator，選邊啲 AVS、配置幾多、allocation delay 係幾長，全部係會直接打中客戶本金嘅風控決策，唔係純粹多賺幾點 yield。

## 小練習（題目 + 參考答案）

**題目：** 產品上線「ETH 再質押活期，隨時可贖」。後台把用戶 10,000 ETH 經 EigenPod 再質押。週一有 2,000 ETH 贖回：job 對住預估金額 `queueWithdrawals`，立刻把 2,000 從「已質押」撥去「可用」並讓用戶提現（用熱錢包墊付）。週三該 Operator 被某個 redistributable AVS slash 一截。第 15 日 `completeQueuedWithdrawal`（`receiveAsTokens: true`）只收到 1,940 ETH。另有一筆 complete 誤把 `receiveAsTokens` 設成 `false`。有咩問題？點改？

**參考答案：**

- **帳本**：queue 成功只可以入「提取中／在途」，**唔可以**加可用。即時墊付若業務堅持要做，必須單獨記一筆平台墊付負債，額度受 slashing haircut 同 14 日鎖限制約，而唔係假設 2,000 會原數回來。
- **罰沒**：隊列中仍然 slashable。到帳以 complete 後托管地址實收 1,940 為準；60 ETH 缺口由墊付賬或產品風險準備金承擔，唔好 silently 當手續費或等下期 rebase 填。
- **原生 ETH**：還要確認退出、sweep、checkpoint 已完成，否則 complete 會失敗；估大／估細都要有剩餘資金同重排隊流程。
- **`receiveAsTokens: false`**：呢筆其實重新 restake 咗，錢冇入熱錢包。監控 complete 回執同餘額差，發現冇到帳要告警，唔好把狀態標成「已贖回」。
- **回歸**：模擬「queue 後、complete 前被 slash」同「testnet 25 blocks 誤當生產延遲」——任何把排隊中金額當可提現嘅路徑都應該失敗。
