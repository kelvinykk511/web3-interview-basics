# 交易所 ETH 質押產品後台

## 人話

用戶撳「質押 ETH」，後台將 ETH 存入 Beacon deposit contract 開 validator；入要排激活隊，出要排 exit 隊再等 sweep。獎勵唔係 transaction，係共識層直接加餘額。

## 面試短答

- 每個 validator 最少 **32 ETH** 先可以激活；**Pectra** 後 **0x02 compounding credentials** 令 **effective balance** 上限去到 **2048 ETH**（**EIP-7251**），可以少開 validator。
- **withdrawal credentials** 決定錢退去邊（**0x01／0x02** 指向執行層地址，應為交易所控制嘅冷錢包／合約），設錯好難救。
- **生命週期**：deposit → 激活隊列 → active → exit 隊列 → withdrawable 延遲（**256 epochs ≈ 27 小時**）→ sweep 自動打款。
- 本金／獎勵經 **EIP-4895 withdrawals** 打入地址：**冇 tx hash、冇 Transfer event** → 掃 tx 嘅 scanner 會漏，要讀 `block.withdrawals` 或餘額差。
- 帳本分「**可用 / 已質押 / 贖回中**」，贖回顯示預計到帳時間或用流動性 buffer 墊付。
- 最大風險係 **slashing**：同一 key 兩部機同時簽（failover 做錯）；要 **slashing protection DB**（**EIP-3076** interchange 格式），HA 用 **active-passive + 鎖**，寧可 offline（小罰）都唔好雙簽。

## 常見追問（連答案）

**Q：(a) 點解寧慢勿雙簽？**  
**A：** offline 罰款大約等於錯過嘅收益；**雙簽 = slashing** 即時罰 + 強制退出 + **相關性罰款**（同期被 slash 人越多罰越重）。

**Q：(b) 獎勵點分俾用戶？**  
**A：** 用 **beacon API** 讀 validator balance 做 **accrual 快照**，partial withdrawal 到帳時同 accrual 對帳；用戶按份額（類似基金 **NAV／share**）分配，唔好一人一個 validator。

**Q：(c) 用戶要即刻贖回？**  
**A：** exit queue 時間不可控（擁擠時可以幾日至幾星期），選擇：顯示 **T+N** 預計時間／自有流動性墊付（有成本同風險）／發 **LST** 俾用戶喺二級市場賣。

**Q：(d) EIP-7002？**  
**A：** Pectra 後可由 **withdrawal credential** 嘅執行層地址觸發 exit，唔一定要 validator key 簽 voluntary exit——託管上持有提款地址嘅一方都可以觸發退出。

## 小練習

**題：** 對帳發現冷錢包 ETH 多咗 **0.0347 ETH** 但 indexer 冇入帳 tx，點查？

**參考答案：** 先查區塊 **withdrawals** 列表有冇打去呢個地址（partial reward sweep，對 validator index）；其次查係咪 **fee recipient** 收 proposer 小費，或 **selfdestruct** 強制轉帳。歸類為質押收益入「自有／待分配」，唔好當無主充值。
