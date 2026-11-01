# EVM 兼容鏈接入（BSC／Polygon／Avalanche 等）：「同 ETH 一樣」嘅陷阱

## 人話
好多鏈話自己「EVM 兼容」：地址格式一樣（0x…）、ERC-20 一樣、JSON-RPC 一樣，所以團隊會想「抄 ETH scanner 改個 RPC URL 就上線」。但係細節唔同，正正係出事位：**最終性、RPC 限制、gas 幣、chainId、合約地址唔同**。

CEX 比喻：好似接多一間「用同一套 API 規格」嘅支付渠道，但佢對賬時間、退款規則、限流全部唔同——你唔可以假設同主渠道一樣。

## 面試短答
接入新 EVM 鏈我會逐項核對：(1) 最終性模型——用 `finalized`／`safe` tag 定係確認數，歷史上有冇深 reorg；(2) RPC 能力——`eth_getLogs` 區塊範圍／結果數上限、有冇 trace/debug API、archive；(3) 鏈參數——chainId、原生 gas 幣、EIP-1559 支援與否、出塊時間；(4) 資產——同一個代幣喺唔同鏈係唔同合約地址、唔同 decimals（例如有鏈上嘅 USDT 係 18 位 decimals），要按 (chainId, contract) 做白名單；(5) 帳本——每條鏈獨立記帳、獨立熱錢包 nonce、獨立對帳。

## 常見追問（連答案）
**Q1：同一個 0x 地址喺 ETH 同 BSC 係咪同一個人？**
A：如果係 EOA（私鑰控制），同一私鑰喺所有 EVM 鏈地址一樣，所以用戶「揀錯鏈」打過嚟，交易所理論上有能力救（見 wrong-chain-token-recovery）。但**合約地址**唔一定：Safe／合約錢包喺另一條鏈可能未部署或者係另一個人部署，出金去合約地址要逐鏈確認。

**Q2：Polygon PoS 試過深 reorg，點影響設計？**
A：2023 年 Polygon PoS 出過一次過百個 block 嘅 reorg，好多只等幾個確認嘅服務中招。教訓：唔好照抄 ETH 確認數；優先用鏈提供嘅 finality（例如支援 `finalized` tag 就等 finalized），冇嘅話按歷史 reorg 深度 + 安全餘量定，同埋 scanner 一定要支援 reorg 回滾。

**Q3：BSC 有 fast finality，可唔可以好快入帳？**
A：BSC 引入咗 validator 投票嘅 fast finality，正常情況幾個 block 就 finalized；入帳可以用 `finalized` tag 而唔係死數確認數。但要監控「finalized 高度停咗唔郁」（投票唔夠時會退化），停咗就要 fallback 到保守確認數或暫停。

**Q4：`eth_getLogs` 喺新鏈成日 timeout／報錯？**
A：各家 RPC 對 block range（例如只准幾千塊）同返回條數有上限，出塊快嘅鏈（1–2 秒一塊）一日塊數多好多。scanner 要可配置每鏈 range、自動對半拆分重試、游標持久化；必要時改用 `eth_getBlockReceipts` 或自建節點。

**Q5：gas 幣唔係 ETH 會影響乜？**
A：熱錢包要另外補 BNB/POL/AVAX 等做 gas；歸集 ERC-20 前要先打 gas 幣入充值地址；出金手續費報價用嗰條鏈嘅原生幣價；部分鏈唔支援 EIP-1559 或者 base fee 行為唔同，fee 策略要逐鏈配置。

## 小練習（題目 + 參考答案）
**題：** 產品想下星期上線「USDT on 某新 EVM 鏈」充提，開發話「改 config 加 RPC URL 同 USDT 合約地址就得」。你列出上線前一定要做嘅 checklist（至少 6 項）。

**參考答案：**
1. 確認 chainId，簽名用 EIP-155 帶 chainId，防重放到其他鏈。
2. 確認 USDT 官方合約地址（官方渠道交叉核對）同 decimals，唔可以假設 6 位。
3. 最終性：有冇 `finalized` tag、歷史最大 reorg 深度，定入帳規則 + scanner reorg 回滾測試。
4. RPC：getLogs range/條數限制、rate limit、至少兩個獨立 RPC 供應商 + 自建節點計劃。
5. gas：原生 gas 幣上架熱錢包、補 gas 流程、EIP-1559 支援與否、手續費報價來源。
6. 熱錢包：獨立私鑰/MPC 配置、獨立 nonce 管理、限額同水位。
7. 帳本同對帳：資產按 (chainId, contract) 唯一，新鏈加入每日鏈上 vs 帳本對帳。
8. 風控：KYT 供應商支唔支援呢條鏈；充值頁清楚顯示網絡名，減少揀錯鏈。
9. 停擺 playbook：鏈停擺／升級時暫停開關。
