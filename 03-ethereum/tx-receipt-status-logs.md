# Transaction Receipt（status／logs／gasUsed）

## 人話（從你熟的 CEX）

廣播之後你只係有咗 `txHash`——好似落單號。真正「成唔成、轉咗幾多、事件係咩」要等節點出 **receipt**。  
CEX 入帳：掃到 Transfer 之後仍要對 receipt（同確認數）；出金：對帳、退手續費、解凍用戶額度都靠 receipt 字段，唔靠「我哋發送成功」。

## 面試短答

- **Receipt 關鍵字段**：`status`（1 成功／0 失敗，Byzantium 後）、`blockNumber`／`blockHash`、`gasUsed`、`effectiveGasPrice`（EIP-1559）、`logs`／`logsBloom`、`contractAddress`（合約創建時）。
- **status=1 ≠ 業務對**：合約可以「成功結束」但冇你要嘅 Transfer（例如內部提前 return）；CEX 入帳要 **parse 對應 Transfer log**（或 balanceOf 前後差），唔好只看 status。
- **status=0**：執行失敗（revert／OOG）；**nonce 已消耗、gas 已扣**；要有失敗單流同埋可選的自動／人工重試（新 nonce）。
- **確認數**：receipt 出現只係「入咗某塊」；CEX 仍用 `tipHeight - txHeight + 1`（或 finality tag）先入帳／放行提現完成。

## 常見追問（連答案）

**Q：pending 時有冇 receipt？**  
A：冇（或客戶端當 null）。`eth_getTransactionReceipt` 回 null＝未上鏈或已被替換／丟棄。要配合 `eth_getTransactionByHash`（看係咪仲 pending）同 stuck-nonce 流程。

**Q：logs 點用喺 ERC-20 充值？**  
A：receipt.logs 裡揾 `Transfer(address,address,uint256)`，topic0＝事件簽名 keccak；`from`／`to` 多在 topic（indexed），`value` 在 data。核對 `to`＝用戶充值地址、`address`＝白名單 token 合約。唔要只信 indexer 推送，要可從 receipt 重放驗證。

**Q：effectiveGasPrice／gasUsed 對 CEX 帳務有咩用？**  
A：實際手續費 ≈ `gasUsed * effectiveGasPrice`（L2 可能另有 L1 data fee）。用嚟：熱錢包成本核算、向用戶收 withdrawal fee 對帳、偵測異常高 gas（被夾／估錯）。

**Q：reorg 之後 receipt 會點？**  
A：原 `blockHash` 可能唔再 canonical；同一 txHash 可能進新塊或暫時消失。入帳水位要用「可逆確認」設計（見 reorg／finality 溫習）：降確認時凍結或回滾未達標分錄。

## 小練習

**題：** 提現單已廣播得 txHash。輪詢發現 receipt.status=0，logs 空。你嘅後端下一步？用戶餘額同 nonce 點處理？

**參考答案：**  
1）確認係真失敗（多節點／同一 blockHash 一致），解碼 revert（trace／call 重放）寫入失敗原因。  
2）提現單 → `ONCHAIN_FAILED`；視產品：自動用 **新 nonce** 重打（要防雙花／幂等鍵），或人工審核。  
3）用戶：若廣播前已鎖額，失敗應 **解凍／退回可用**（唔好當成功扣帳）。  
4）熱錢包 nonce：該 nonce 已耗盡，下一筆要用 nonce+1；記成本 gas。  
5）告警：連續 status=0 暫停該鏈自動出金。
