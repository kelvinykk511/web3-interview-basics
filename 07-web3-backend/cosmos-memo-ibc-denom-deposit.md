# Cosmos 系充值：memo + IBC denom + relayer 陷阱

## 人話
Cosmos（ATOM、OSMO 等）係帳戶模型，同 XRPL 一樣交易所通常用「一個共用充值地址 + memo」分用戶。最大坑係 IBC：跨鏈過嚟嘅幣，名唔叫 USDC，叫 `ibc/<hash>`，而且帶 memo 嘅位置唔同。

## 面試短答
1) 共用地址 + memo 識別用戶，memo 錯/漏 → 入無主充值走找回流程。2) 幣種用 **精確 denom** 做白名單，唔用 symbol：IBC denom = `ibc/` + SHA256(`transfer/channel-X/基礎denom`)，同一個 USDC 經唔同 channel/路徑過嚟就係唔同 denom，風險同價值都可能唔同。3) 入帳睇 tx `code == 0` 同 events（`coin_received`/`transfer`），唔好只 parse 頂層 `MsgSend`，因為有 multi-msg、`MsgExec`（authz）、`MsgMultiSend`。4) CometBFT 係即時最終性，commit 咗唔會 reorg，但仍要用自己可信節點、等少量 block 防節點出錯。5) 失敗 tx（code≠0）一樣上鏈收 fee，唔入帳。

## 常見追問（連答案）
- **IBC 充值點解喺 tx memo 搵唔到用戶 memo？** 收款鏈上嗰條 tx 係 relayer 提交嘅 `MsgRecvPacket`，tx memo 屬於 relayer。用戶 memo 喺 ICS-20 packet data 入面（`receiver`/`denom`/`amount`/`memo`），要 decode packet 先攞到。
- **IBC 充值點先算成功？** `MsgRecvPacket` tx code=0 之外，要睇 `write_acknowledgement` 嘅 ack 冇 error（ack error 代表冇 mint/解鎖，來源鏈會退款）；入帳金額以 `coin_received`/`fungible_token_packet` event 為準。
- **ATOM 經 Osmosis 打返去 Cosmos Hub 係咩 denom？** 回家路徑會「剝殼」，到 Hub 變返原生 `uatom`；但 Osmosis 原生 OSMO 去 Hub 就係 `ibc/...`。所以要按 (鏈, 精確 denom) 映射到內部幣種。
- **出金經 IBC 有咩要處理？** IBC transfer 可能 timeout（timeout height/timestamp），之後退返交易所熱錢包；狀態機要有 `sent → acked / timeout_refunded`，refund 唔可以當成新充值或者遺漏，要對返原出金單退返用戶。
- **出金 nonce 點處理？** Cosmos 用 `account_number + sequence`，同 EVM nonce 一樣要單調、單一 signer 串行；gas 用 simulate × adjustment（例如 1.3），fee 按 denom 最小單位（`uatom` = 10^-6 ATOM）。
- **memo 有長度限制？** 有，鏈參數 `max_memo_characters`（Hub 預設 256），memo 格式要校驗。

## 小練習（附參考答案）
**題：** 用戶由 Osmosis 經 IBC 打 USDC 入交易所 Cosmos 地址，填咗 memo，但掃描器報「無 memo 無主充值」，而且幣種識別成未知 `ibc/ABC...`。點排查同修？
**參考答案：** 兩個問題。1) memo：掃描器讀咗 relayer 嘅 tx memo；應 decode `MsgRecvPacket` 嘅 ICS-20 packet data 攞 `receiver` + `memo`，並確認 ack 成功。2) 幣種：`ibc/ABC...` 唔喺白名單，要追 denom trace（`/ibc/apps/transfer/v1/denom_traces/{hash}`）睇路徑同基礎 denom，確認係咪我哋支援嘅 USDC 來源（例如 Noble 原生 USDC 經指定 channel）；係就加 (鏈, denom) 映射，唔係就走找回/退款流程。修完用 txhash 冪等重掃補入帳。
