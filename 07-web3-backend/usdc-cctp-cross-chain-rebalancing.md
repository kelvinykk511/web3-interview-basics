# 原生 USDC 跨鏈（CCTP）vs 橋接 USDC.e

## 人話

同叫 USDC，Circle 原生發行嘅同橋鎖倉 mint 出嚟嘅 **USDC.e** 係兩隻唔同合約。**CCTP** 係 Circle 官方「燒咗再印」：源鏈 **burn** → Circle **attestation** 簽名 → 目標鏈 **mint**；冇流動性池、冇 wrapped token。

## 面試短答

- 上架／帳本用 **(chainId, contract)** 做 key，USDC 同 USDC.e **分開幣種**或明確 mapping，USDC.e 帶橋風險。
- 交易所用 CCTP 做跨鏈熱錢包調撥，狀態機：**BURN_SENT → BURN_FINAL → ATTESTATION_READY → MINT_SENT → MINT_CONFIRMED**。
- message 由 **(source domain, nonce)** 唯一識別，目標鏈 **MessageTransmitter.receiveMessage** 同一 message **只可用一次**（重複會 revert），自己 DB 亦以 **(sourceDomain, nonce)** 去重。
- 任何人都可以 call **receiveMessage**（relayer），錢仍去 **mintRecipient**；可設 **destinationCaller** 限制。
- **Standard Transfer** 等源鏈 finality（Ethereum 大約十幾分鐘）；**V2 Fast Transfer** 用軟確認快好多但收費。
- **mintRecipient** 係 **bytes32**（地址左補零），填錯 = mint 去錯地方。

## 常見追問（連答案）

**Q：(a) burn 咗未 mint 錢喺邊？**  
**A：** 係**在途資產**，帳面記「跨鏈在途」，對帳要計入；拎到 attestation 就可以由補做 job 去 mint。

**Q：(b) mint tx 失敗／卡住？**  
**A：** 用同一 **message + attestation** 重提；如果 revert 原因係 **nonce 已用** = 其實已 mint，查事件確認，**絕對唔好**當失敗再 burn。

**Q：(c) 點解唔用第三方橋？**  
**A：** 鎖倉橋有合約／驗證者被黑風險（歷史上多宗大額橋被盜），拎到嘅係 wrapped token；CCTP 冇池、原生 burn/mint，但依賴 **Circle attestation 服務**（中心化，佢慢／掛就延遲）。

## 小練習

**題：** 調撥 50 萬 USDC 由 Ethereum 去 Base，burn 成功 20 分鐘後 attestation 仲 pending，worker 重試咗 burn。出咗咩事？點防？

**參考答案：** burn 咗兩次 = **100 萬在途**（錢唔會丟，兩筆都 mint 得返，但熱錢包流動性被鎖）。根因係「發 burn」同「等 attestation」冇分開狀態。防法：一張調撥單只准一個 burn（**BURN_SENT** 之後禁止再發），之後只以 tx hash poll attestation，超時告警唔自動重 burn；善後係補 mint 第二筆再反向調撥。
