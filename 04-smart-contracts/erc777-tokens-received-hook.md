# ERC-777 tokensReceived Hook（充值／轉帳回調風險）

## 人話（從你熟的 CEX）

ERC-20 嘅 `transfer` 對收款方嚟講多係「錢入咗帳，對方合約唔會被叫醒」。  
ERC-777（同部分有 hook 嘅 token）轉帳成功前後可以 **callback 收款合約**（`tokensReceived`；類似 ERC-1363、ERC-1155 `onERC1155Received`）。類比：你以為只係 DB 加餘額，其實對方帳戶觸發咗 webhook——對方合約可以喺你哋入帳邏輯中途再入、再轉、重入。

CEX 充值地址若係 **智能合約**（CREATE2 deposit、smart wallet、錯誤部署嘅 forwarder），收 ERC-777 可能喺 tx 中間跑陌生 code。EOA 充值地址冇 code，收唔到呢類 hook——但歸集目標、流動性合約、自建 vault 仍要防。

## 面試短答

- **ERC-777**：可選 operator；`send`／`transfer` 路徑上對持有人／收款人註冊嘅 **tokensToSend／tokensReceived** hooks（經 ERC-1820 registry）。收款方係合約且註冊咗 implementer，就會被 callback。
- **風險**：重入（hook 內再轉出／再觸發你哋入帳）、Gas 耗盡／故意 revert 導致轉入失敗、假進度（你以為 credit 已寫但外層 revert）、同「先 effects 再互動」紀律衝突。
- **CEX 實務**：
  - 充值掃描：**唔好假設只有 ERC-20 Transfer**；上幣白名單審核要標「有 receiver hook／唔標準」。
  - 入帳狀態機：先落「鏈上觀察」再異步入帳；合約內若要處理收款，遵循 checks-effects-interactions 或 reentrancy lock；最好充值地址用 **EOA／無 hook 路徑**。
  - 歸集／出金：對可疑 token 用經審核嘅路徑；留意有啲 777 上 `transfer` 仍會 hook。
- **親戚**：ERC-721/1155 safeTransfer 嘅 `onERC721Received`／`onERC1155Received`、ERC-1363——同一類「轉帳叫醒對方」。

## 常見追問（連答案）

**Q：用戶提到 EOA 充值地址，仲使唔使怕 tokensReceived？**  
A：該筆 **轉入 EOA** 唔會執行 tokensReceived（無 code）。風險轉去：(1) 你哋 **sweep 到合約金庫** 嗰下；(2) 熱錢包／custody 合約收幣；(3) 惡意 token 喺 transfer 過程搞 sender hook（`tokensToSend`）搞到歸集失敗。上幣同歸集路徑都要審。

**Q：點樣喺上幣審核發現？**  
A：睇是否 ERC-777／1820 implementer、有冇 `tokensReceived`、transfer 會唔會外部 call；用 fork 對測試收款合約轉一筆睇 call trace；查知名攻擊模式（hook 重入）。唔喺白名單就唔好開充值。

**Q：同「假 Transfer log／地址投毒」點分工？**  
A：投毒／假 event 係 **索引／展示層** 被騙；777 hook 係 **執行層** 喺同一筆 tx 被回調。防禦前者靠白名單＋value 門檻＋完整地址；防禦後者靠唔用可重入收款合約、CEI、鎖、同審核 token 行為。

**Q：面試官問：deposit contract 點寫先穩？**  
A：簡潔答：充值地址優先 EOA；若必須合約收款——`tokensReceived`／`onERC*` 只做最小記錄（emit + 記額），**禁止** 喺 hook 入面再外部 call 或即時 credit 可提現餘額；對外信用放在確認數達標嘅異步服務。ReentrancyGuard + 白名單 token。

## 小練習

**題：** 你們新建一個 CREATE2 充值合約，收到 token 就即時 `credit(user)`。有人轉入一隻註冊咗 tokensReceived 嘅 ERC-777，hook 入面再呼叫你哋嘅 `credit` 或再轉出。會出咩問題？點改架構？

**參考答案：**  
問題：hook／重入可令 `credit` 被呼叫多次、或 credit 後再被轉走導致「帳多幣少」；若你喺 hook 同步寫咗 DB（跨系統），外層 revert 仲可能令鏈上回滾但 DB 唔滾。  
改法：(1) 充值地址改 EOA，或合約 hook **唔呼叫**任何入帳；(2) 只 emit 標準事件，由索引服務等 N 確認後入帳；(3) 合約若必須實作 `tokensReceived`，只寫 storage 標記 + 防重入，**禁止**再外部 call；(4) 上幣拒絕未審 777／有任意 callback 嘅 token。核心：鏈上觀察同帳本 credit 解耦，同 CEX 充值流水一致。
