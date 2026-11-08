# ERC-4337 Session key：出金／操作權限邊界

> 銜接：Paymaster 贊助見 [`erc4337-paymaster-sponsorship-quota.md`](./erc4337-paymaster-sponsorship-quota.md)。Session key **唔係** ERC-4337 規範嘅一部分，而係 Smart Account（模組／plugin，好似 ERC-7579）自己做嘅「有限授權簽名者」。

## 人話（從你熟的 CEX）

CEX API key 你一定識：只讀、可交易、可出金係分級嘅；出金 key 通常仲有 IP 白名單、地址簿、日限額，撤銷時唔使換主密碼。  
**Session key** 就係 Smart Account 版嘅 API key：owner 主鑰可以繼續放硬件錢包／MPC，另外發出一把短命、權限收窄嘅鑰，專俾 App 代簽某啲 UserOp。`validateUserOp` 見到呢把鑰，只會接受政策允許嘅 `callData`；超出範圍就當無效簽名。

對 CEX 最有用嘅兩種政策：
1. **充值 session**：只准把指定代幣轉去「呢個用戶自己嘅 CEX 充值地址」，額度同期限都鎖死。
2. **出金 session**（若真係要做）：只准轉去地址簿入面已冷卻完成嘅地址，仲有日限額——而且好多團隊寧願出金仍然要 owner 親簽，根本唔發長命出金 session。

政策一闊（例如「允許帳戶嘅任意 `execute()`，有效期 30 日」），失竊就等如交出主鑰。

## 面試短答

- **誰驗**：EntryPoint 叫帳戶 `validateUserOp`。帳戶邏輯：owner 簽名 → 全權；session 簽名 → **對照政策**（時間窗、目標合約、selector、金額／累計額度、收款地址）。唔通過就當驗證失敗，Bundler／EntryPoint 唔會執行。
- **Default-deny**：沒寫明允許嘅一律拒絕。Admin 類永遠唔准 session 做：換 owner、upgrade 實現、加／改／刪 session、開新 module、任意 `delegatecall`。
- **`executeBatch` 要逐筆驗**：只檢查第一下 call 係漏洞。攻擊者會把「轉去白名單」同「`approve(黑客, max)`」捆在同一 batch。
- **`approve` 好危險**：一次無限授權，之後唔使再經過你條 session 政策都可以被 spender 抽走。充值型 session 應該 **禁止 approve**，或者只准對已知 router 授很小、有期限嘅額度。
- **包裝攻擊**：看似允許嘅 Multicall／換匯／callback（ERC-777 tokensReceived、 DEX callback）入面藏住轉走資金。驗證要解到最內層 call，唔可以只睇外層 selector。
- **額度**：單筆＋窗口累計；原生幣睇 `value`，代幣最好做 **per-token cap**（避免用可操縱 oracle 折算 USD）。用完或者過期即失效。
- **撤銷**：owner 可以即時 disable；App 若係發行方，要有一鍵撤銷同「裝置遺失」流程，好似撤 API key。
- **對 CEX 充提掃描**：用戶用 session 轉幣入你地址，入帳仍然只睇 `Transfer`／native value（尋日結論不變）。Session 影響嘅係 **用戶側授權模型**，同你 indexer 無關；但若你自研 AA 錢包／App 代簽，政策錯就是客戶資產事故。

## 常見追問（連答案）

**Q1：理想嘅「一鍵充值到本所」session 應該點寫？**  
A：只允許 `transfer`／帳戶 `execute` 到 **該用戶獨一無二嘅充值地址**（bound `userId + chainId + asset`）；token 白名單（例如 USDT／USDC）；單筆同每日上限；短 TTL（數小時到一日，到期再續）；禁止 approve、禁止改 session、禁止其他目標。即使手機連 session 一齊被偷，攻擊者都只能把幣打入被害人自己嘅 CEX 帳戶，之後仲要過你哋登入／2FA／出金風控先提走。

**Q2：出金（從用戶 AA 轉走法幣世界）可唔可以發一把長命 session？**  
A：極不建議長命、廣權限。若產品堅持「細額免 confirm」，也要：地址簿＋新增地址冷靜期、日限額、短 TTL、禁止 admin／approve、每筆仍可設次級確認（大額升級到 owner）。主鑰／監護人永遠保留緊急撤銷。面試可以講：出金 session ≈ 帶提現權限嘅 API key，你哋自己都唔會發「無限額、任意地址、30 日有效」嘅提現 key。

**Q3：Session key 同 Paymaster、owner key 點分工？**  
A：Owner = root（恢復、升級、發／撤 session）。Session = 日常有限操作。Paymaster = 邊個付 gas，**唔改變** 操作權限——贊助一筆越權 UserOp 仍然應該喺 `validateUserOp` 被帳戶拒絕。安全要兩層都收緊：帳戶政策拒越權，Paymaster allowlist 拒贊助危險 selector。

**Q4：點解「允許 `execute(address,bytes)`」幾乎等如交出錢包？**  
A：`execute` 嘅參數就係任意目標＋任意 calldata，等於讓 session 以帳戶身分做任何事：轉光代幣、`approve` 黑客、甚至呼叫自己嘅 admin 函數（若未再包一層檢查）。面試要講：允許嘅係 **解碼之後** 的具體 selector＋目標＋參數約束，而唔係允許呢個通用入口本身。

**Q5：CEX 出金到用戶 AA 地址，要唔要理佢有冇 session key？**  
A：一般唔使。你出金只保證打到用戶提供而且通過校驗嘅合約地址（見 [`eth-withdrawal-to-contract-wallet.md`](./eth-withdrawal-to-contract-wallet.md)）。對方內部邊把鑰可以再轉走，係用戶錢包風險，唔係你廣播路徑要解析的東西。你要防嘅係打錯地址、合約唔收 ETH、以及用戶被釣魚改地址簿。

## 小練習（題目 + 參考答案）

**題目**：錢包 App 想用 session key 減少彈窗。PM 話「直接允許帳戶嘅 `execute()`，有效期 30 日，最順」。上線後用戶報失手機，鏈上見到 USDT `approve` 給未知 spender，隨後餘額被 `transferFrom` 抽走。缺咗咩邊界？「只准充值到本所」嘅 session 應該點寫？若仲想做細額出金呢？

**參考答案**：
- **事故根因**：無限制 `execute`＋長 TTL = owner 等價。`approve` 唔受單筆轉帳額度約束，一次授權就繞過之後所有 session 檢查。
- **必補邊界**：default-deny；禁止 admin／upgrade／改 session／`delegatecall`；`executeBatch` 同 callback／Multicall **逐層解碼**；原則禁止 `approve`（或只准白名單 spender＋有上限＋短許可期限）；per-token 額度；可即時撤銷。
- **充值 session**：目標鎖定該用戶 CEX 充值地址；只准特定 token 的 `transfer`；短 TTL＋每日上限。失竊時錢仍然入用戶自己嘅交易所帳戶。
- **細額出金 session**（可選、更嚴）：只准地址簿且過冷靜期嘅目的地、更低日限、更短 TTL、大額強制 owner 簽；唔好發 30 日萬能鑰。
- **回歸用例**：充值到自己嘅 CEX 地址成功；改 `to` 去另一個地址失敗；batch 裏夾帶 approve 失敗；過期 session 失敗；owner 撤銷後即時失效。
