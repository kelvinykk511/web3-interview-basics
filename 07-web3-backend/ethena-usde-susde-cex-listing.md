# Ethena USDe／sUSDe：CEX 上架、入帳同庫存（唔好當成 USDC）

> 銜接：stETH rebase vs wstETH 見 [lst-rebasing-steth-wsteth.md](./lst-rebasing-steth-wsteth.md)，ERC-4626 份額記帳見 [erc4626-vault-shares-cex.md](./erc4626-vault-shares-cex.md)，穩定幣凍結見 [stablecoin-blacklist-freeze.md](./stablecoin-blacklist-freeze.md)，上架清單見 [token-listing-onchain-checklist.md](./token-listing-onchain-checklist.md)。今日將呢套用喺 **USDe（合成美元）同 sUSDe（質押份額）**，唔好同 USDC／USDT 混成「都係穩定幣」。

官方說明：[How USDe Works](https://docs.ethena.fi/overview/how-usde-works)、[Minting USDe](https://docs.ethena.fi/technical-design/minting-usde)、[User security measures](https://docs.ethena.fi/technical-design/minting-usde/user-security-measures)、[Staking USDe](https://docs.ethena.fi/technical-design/staking-usde)、[Staking key functions](https://docs.ethena.fi/technical-design/staking-usde/staking-key-functions)、[FAQ](https://docs.ethena.fi/resources/faq)。冷卻參數會變，以鏈上 `cooldownDuration` 為準。

## 人話（從你熟的 CEX）

USDe **唔係** Circles／Tether 那種用法幣儲備支撐嘅穩定幣。官方定位係 **synthetic dollar**：組合現貨（或代幣化資產）同大約等額嘅空頭期貨／永續對沖，再加一部分本身已穩定嘅資產（穩定幣、短久期 RWA），令組合盡量 **delta-neutral**，再以呢個組合作為發行 USDe 嘅 backing。風險嚟自對沖場所、funding／基差、托管同「Off Exchange Settlement」、以及二級市場價格可以同「目標 1 美元」分開走。

鑄幣／贖回亦唔係人人可按 1:1 隨時兌。只有通過 KYC／AML、被協議 **whitelist** 嘅地址，先可以用 Pricing API ＋ EIP-712 訂單，原子地 mint／redeem；Ethena 可以拒單，`GATEKEEPER_ROLE` 仲可以喺異常時 **即時關閉** 鑄贖。未入白名單嘅人（同好多時候交易所自己）只可以喺 CEX／DEX 二級市場買賣。所以「庫存隨時可以按協議兌回 USDT／USDC」呢句，未做完對手方准入同配額之前，唔成立。

sUSDe 係把 USDe 存入 StakedUSDe 之後拿到嘅 **ERC-4626 份額**。USDe 本身唔生息；獎勵以 USDe 轉入金庫，份額匯率向上（或持平）。負收益官方寫由 Reserve Fund 吸收，唔會從金庫扣 USDe。解質押唔等於即時到帳：要先申請，USDe 進入 **USDeSilo**，過咗 **cooldown** 先 withdraw。合約 admin 可 `setCooldownDuration`（官方上限 90 日）。另有按 backing 流動性調整嘅動態框架；Ethena Foundation 2026 年 7–9 月治理更新寫當時仍係 1 日——但呢個會變，後台要讀鏈上現值，唔好寫死。

## 面試短答

1. **USDe ≠ USDC／USDT，風控唔能當現金等價。** 入帳可以當普通 ERC-20 掃 `Transfer`，但估值、保證金、內部 1:1 兌 USD／USDT 要獨立 haircut 同市場價，唔好用「協議目標釘住」當標記價。鑄贖窗口唔等於二級流動性。
2. **直接 mint／redeem 係許可制。** 只有白名單地址做得到；報價有滑點區間，成交要落喺用戶簽過嘅範圍。Gatekeeper 可停鑄贖。庫存出口要同時計：自身是否已准入、配額／拒單、以及二級市場深度。唔好把「協議設計可以 on demand 贖回」寫成出金 SLA。
3. **USDe 同 sUSDe 係兩個 asset_id。** sUSDe 係份額，枚數唔 rebase（對比 stETH）；匯率隨已歸屬獎勵上升。入帳同負債以 sUSDe 枚數為準，再用匯率換算展示，唔好把充值到嘅 sUSDe 1:1 記成可用 USDe。獎勵大約每 8 小時注入、再線性歸屬 8 小時，避免瞬間拉高匯率被 sandwich。
4. **解質押有狀態機：request → cooldown（Silo）→ withdraw。** 「冇最短質押期」只係話你可以 stake 完下一塊就申請 unstake，**唔代表**即刻收到 USDe。Cooldown 讀合約參數。等待期間用戶可以喺二級市場賣 sUSDe，但可能折價，協議唔保證有盤。Earn 產品若幫客 stake，就要暴露到帳時間，禁止用尚未冷却完嘅資產去墊付即時 USDe 出金。
5. **sUSDe 有合規凍結／限制。** `Blacklister` 可把地址設成 fully restricted（官方：收唔到 sUSDe），admin 甚至可以把受限地址裡鎖定嘅 sUSDe **reallocate** 走（官方寫類似 USDC 合規機制，並聲明只限 sUSDe）。充提要監控呢類事件，當托管風險。官方仲寫：sUSDe 唔向歐盟／EEA 居民提供取得；FAQ 寫美國用戶唔能用其 mint 應用——上架同 Earn 要過合規，唔只過技術 checklist。

## 常見追問（連答案）

**Q1：官方話 stake 冇最短期限，點解又有 cooldown？**  
A：兩件唔同嘅事。冇最短期限＝唔使先鎖倉 N 日先有資格申請贖回。Cooldown＝申請之後，要等 `cooldownDuration` 先可以从 Silo 提取 USDe。面試如果只答其中一句，會被追問「咁客戶而家按提款，幾耐到帳」。

**Q2：客戶把 USDe 當保證金，或者產品寫「1:1 美元穩定幣」，有咩問題？**  
A：Backing 係 delta-neutral 組合，唔係銀行現金；二級市場可以脫錨；鑄贖只對白名單開放，而且可以被拒或被 gatekeeper 關掉。風控應當風險資產：獨立標記價、haircut、集中度限額。出金 USDe 本身仍係 ERC-20 轉帳；困難喺「保證隨時兌成法幣或 USDT」。

**Q3：我哋做 Earn，自動幫客把 USDe stake 成 sUSDe，客戶要即提 USDe，可唔可以先由熱錢包墊付？**  
A：若 cooldown 未完就墊付，等於交易所用自有 USDe 庫存借俾客戶，期間承擔協議、折價同鑄贖關閉風險。要先產品化：分開帳本（USDe／sUSDe／cooling）、展示等待時間、或者讓客戶喺現貨賣 sUSDe（可能滑點）。只有庫存同風險限額明確允許先墊，先可以做，而且唔能假定協議 1:1 即時贖回嚟平倉。

## 小練習（題目 + 參考答案）

**題目：** 產品要把 USDe 標成「同 USDC 一樣嘅穩定幣」。後台把用戶充入嘅 sUSDe 1:1 記成可用 USDe；Earn 自動 stake，並允許客戶即時提取 USDe（熱錢包墊付）；庫存模型假設隨時可向協議 1:1 贖回。有咩問題？點改？

**參考答案：**

- **定位唔能當 USDC**：披露合成美元同對沖／托管風險；保證金同內部劃轉用獨立標記價＋haircut，唔寫 1:1 現金等價。
- **sUSDe ≠ USDe**：兩個 asset_id；入帳記 sUSDe shares，展示匯率分開；禁止把份額 1:1 撥入 USDe 可用餘額。
- **Earn 即時出金係墊付**：要 request／cooldown／claim 狀態機，讀鏈上冷卻參數（唔寫死 1 日）；即時提款只能賣現貨或動用單獨限額嘅自有庫存，並計入鑄贖被拒／被關嘅情景。
- **贖回假設過強**：未入白名單就冇協議鑄贖；入咗都可能拒單或被 gatekeeper 暫停。庫存要有二級市場深度同應急平倉預案。
- **合規同凍結**：監控 sUSDe 限制／redistribute；熱錢包被限制要當 P0。EU／EEA、美國用戶能否買 sUSDe／用 Earn，交合規定，唔只上架技術。
