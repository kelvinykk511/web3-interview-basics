# Chainlink CCIP 2.0：CEX 金庫跨鏈（同 CCTP／OFT 點分工）

> 銜接：USDC 官方 burn／mint 見 [usdc-cctp-cross-chain-rebalancing.md](./usdc-cctp-cross-chain-rebalancing.md)，Gateway 統一餘額見 [circle-gateway-unified-usdc-balance.md](./circle-gateway-unified-usdc-balance.md)，LayerZero OFT 見 [layerzero-oft-vs-cctp-rebalancing.md](./layerzero-oft-vs-cctp-rebalancing.md)，意圖橋／墊付唔好當成官方路徑見 [erc7683-intent-bridge-cex.md](./erc7683-intent-bridge-cex.md)。今日講 **CCIP**：一條可以同時帶 token 同任意 payload 嘅跨鏈訊息協議，預設用委員會驗證，執行喺目標鏈係許可制開放。

官方說明：[Architecture overview](https://docs.chain.link/ccip/concepts/architecture/overview)、[Message lifecycle](https://docs.chain.link/ccip/concepts/message-lifecycle)、[Best practices (EVM)](https://docs.chain.link/ccip/evm/concepts/best-practices)、[Transfer USDC with Data](https://docs.chain.link/ccip/evm/tutorials/application-developers/usdc)、[Manual execution](https://docs.chain.link/ccip/concepts/manual-execution)。支援鏈、lane、費率、限額以 [CCIP Directory](https://docs.chain.link/ccip/directory) 為準，唔好寫死。

## 人話（從你熟的 CEX）

CCIP 唔係「再一個 USDC 橋」咁簡單。你喺源鏈叫 Router 嘅 `ccipSend`：協議收手續費，token pool 把幣 **lock 或 burn**，然後發出 `CCIPMessageSent`。之後獨立嘅 Cross-Chain Verifier（CCV）等源鏈達到要求嘅最終性，先簽署 attestation。默認每條 lane 都有一個 **Committee Verifier**（官方寫 16 個獨立節點營運方），有需要可以再加 CCV。任何人拿到齊所需證明，都可以喺目標鏈 OffRamp `execute`：校驗證明之後，pool **release 或 mint**，若有 data 就再 callback `ccipReceive`。

類比你哋內部「跨站調撥單」：下單（源鏈 lock／burn）≠ 對方倉庫已入貨（目標鏈 execute 成功）。中間仲有驗證、限額、目標池流動性、同接收合約 gas。RMN 可以對某條鏈或 lane 下 **curse**，期間 send／execute 會 revert——等於風控熔斷，要暫停呢條路嘅調撥。

## 面試短答

1. **源鏈成功 ≠ 目標鏈已到帳。** 入帳／庫存只認目標鏈 execute 成功，以及 pool 嘅 `Released`／`Minted`（或等值事件）同收款地址實收。冪等鍵用 **`messageId`**（官方：`keccak256(encodedMessage)`），唔好用源鏈 tx hash 預入可用餘額。默認執行器對 `FAILURE` 會喺一段窗口內重試（文件寫而家係 8 小時）；之後要 **manual execution**，唔好再 `ccipSend` 一筆。
2. **Pool 機制決定風險模型，唔好全部當成 CCTP。** Burn／Mint 係發行方授權銷毀再鑄；Lock／Release 要靠目標鏈 lockbox 有足夠流動性，唔夠就執行失敗，幣會「在途」直到補流動性再 manual exec。官方仲提醒避免喺多條發行鏈同時用 Lock／Release 造成流動性碎片。CCIP 2.0 訊息 **最多帶一種 token**。`chain selector` ≠ `chainId`（似 LayerZero 嘅 eid）。
3. **最終性預設要等源鏈完整 finality；FTF 要顯式開啟。** Faster-Than-Finality 係 opt-in：訊息發送方（所有訊息）同 token issuer（token 轉帳）都要同意。金庫大額唔好偷偷開 FTF 來追速度。預設路徑會等源鏈最終確定先有 attestation，所以「源鏈 receipt」之後仍然可能要等十幾分鐘（視鏈而定）。
4. **USDC 走 CCIP 時，底層仍然係 Circle CCTP。** 官方 USDC 教程：源池經 CCTP burn，額外嘅 **CCTP Verifier** 核對 Circle attestation，目標池再用呢份證明 mint。所以純 USDC 調撥，CCTP／Gateway 通常更直接；CCIP 適合「幣 + 指令／資料」或非 USDC、官方 lane 已支援嘅資產。唔好對同一筆 burn 同時按 CCTP `receiveMessage` 同 CCIP OffRamp 入帳兩次。
5. **Gas、限額、curse 都要入狀態機。** `extraArgs.gasLimit` 係目標鏈 `ccipReceive` 的上限；純 EOA 收 token、冇 receiver 邏輯就設 `0`。Gas 唔夠或接收合約 revert → `FAILURE`，可加 `gasLimitOverride` 再執行（override 必須 ≥ 訊息裡嘅 limit）。未用完嘅 gas 唔退。Token pool 有 inbound／outbound rate limit。RMN curse 出現就要暫停該 lane，唔好當普通 RPC 故障重試。Router 地址跨升級保持穩定，OnRamp／OffRamp 可以替換。

## 常見追問（連答案）

**Q1：已經有 CCTP 同 LayerZero OFT，仲要 CCIP 做咩？**  
A：分工唔同。USDC 官方 1:1 優先 CCTP／Gateway；某幣官方 OFT 路徑先跟該幣嘅 DVN 棧。CCIP 係通用訊息＋token，帶委員會驗證同可加 CCV，適合要一併傳 data、或多種已支援資產用同一套狀態機。唔好用意圖橋 filler 墊付去替代呢幾條可核對嘅官方／發行方路徑。具體 lane 同限額睇 Directory。

**Q2：目標鏈 execute 失敗，錢係咪沒咗？要唔要再 send 一次？**  
A：通常唔會沒。源鏈已經 lock／burn，訊息有 `messageId`。失敗多數係 dest gas 不足、receiver revert、或 Lock／Release 池流動性不夠。查 execution state：成功過就唔好再執行；`FAILURE`／未成功可以 manual exec（必要時提高 gas，Lock／Release 要先補流動性）。**絕對唔好**再 `ccipSend` 一筆，否則會變成兩次 lock／burn。

**Q3：可唔可以源鏈 `CCIPMessageSent` 就撥目標鏈熱錢包可用？**  
A：唔可以。驗證仲未完成，亦可能要等 finality、可能被 curse、可能 dest 執行失敗。帳本應分「源鏈已鎖／已燒（在途）→ 證明齊備（可執行）→ 目標鏈已 release／mint」。FTF 只係縮短驗證等待，唔改變「要等 dest 成功」。用戶充值如果係經 CCIP 打入你嘅地址，同樣只認目標鏈實收，唔好用源鏈 event 預入。

## 小練習（題目 + 參考答案）

**題目：** 金庫要把 300 萬某治理代幣由 Ethereum 調去 Base。Job 見到 `ccipSend` 成功就即時把 Base 可用庫存 +300 萬。Token 喺 Base 用嘅係 Lock／Release pool。20 分鐘後 dest 執行一直 `FAILURE`，值班同事再對同一筆需求發咗第二次 `ccipSend`。有咩問題？點改？另外，若調撥嘅其實係 USDC，路徑應點揀？

**參考答案：**

- **源鏈成功唔能加目標鏈可用**：先入「CCIP 在途」，用 `messageId` 跟狀態。Dest `FAILURE` 要查原因：若係 lockbox 流動性不足，補流動性再 manual exec；若係 `ccipReceive` gas／revert，用合規嘅 gas override 重試。冇成功 event 同實收之前，Base 可用不能加。
- **唔可以再 send 一次**：第二次會再 lock／burn 300 萬，變成兩筆在途。正確係對原 `messageId` 執行（或等默認 executor 窗口內重試）。兩筆都成功就要反向調返多餘嗰筆。
- **Lock／Release 要監控目標池／lockbox 餘額同 rate limit**，好似監控熱錢包餘額一樣；唔夠流動性係營運問題，唔係「橋壞咗就當丟幣」。
- **若係 USDC**：優先走官方 CCTP 或已完成最終性嘅 Gateway，而唔係為咗單一轉帳再包一層 CCIP（除非你真係要一併傳 data）。若仍然走 CCIP USDC，要明白底層係 CCTP，對帳時唔好同直接 CCTP mint 重複入帳。Curse／Directory 限額變更要納入暫停條件。
