# HD Wallet（BIP-32／BIP-44）— CEX 充值地址派生

## 人話（從你熟的 CEX）

交易所要一次過為好十萬用戶派「專屬充值地址」。如果每條地址都獨立隨機生成私鑰，備份同審計會地獄。  
**HD（Hierarchical Deterministic）錢包**：用一條 **master seed** 按路徑公式派生出成棵地址樹——知道 seed + path 就可以重算出同一條地址同私鑰。

對照你已知嘅 **CREATE2**：CREATE2 係「合約地址公式」（私鑰喺 factory）；HD 係「EOA 私鑰派生」。CEX 兩者都會用：熱／歸集錢包同 per-user EOA 充值多用 HD（或 HSM／MPC 包住 seed）；智能合約收款器多用 CREATE2。

## 面試短答

- **BIP-32**：由 seed → master key，再用 **child index** 派生子私鑰／公鑰；支援 **公鑰派生**（extended public key `xpub`）喺唔暴露私鑰嘅情況下生成收款地址。
- **BIP-44** 路徑慣例：`m / purpose' / coin_type' / account' / change / address_index`  
  例如以太坊常見：`m/44'/60'/0'/0/0`（`60` = ETH coin type；最後一個 index 遞增派用戶地址）。
- CEX 實務：冷端／HSM 保管 seed 或 account key；熱端只持 **有限 range 嘅派生子鑰** 或只用 `xpub` 生成地址做掃鏈；**永不**把整條 master seed 放應用伺服器明文。
- 入帳掃描綁「地址 → userId」映射表；派生要 **可重放**（同一 path 永遠同一地址），方便災難恢復。

## 常見追問（連答案）

**Q：hardened（`'`）同 non-hardened 派生差喺邊？對 CEX 意味咩？**  
A：Hardened 子鑰唔可以由 parent **公鑰**單獨推出（要 parent 私鑰）。Account 層常用 hardened，避免洩漏一條子私鑰 + xpub 就能推兄弟鑰（BIP-32 經典風險）。CEX：對外只發放地址／最多 account 級 `xpub` 畀「只讀掃鏈服務」時，要想清楚層級同洩漏面。

**Q：點解面試會問 xpub？**  
A：因為「熱錢包服務只負責生成充值地址同掃 `Transfer`，唔持有可簽名私鑰」係常見架構：冷端給 `xpub`，熱端派 `address_index`、寫 DB 映射。提現簽名喺另一條 HSM／MPC 路徑，權限分離。

**Q：HD EOA 同 CREATE2 收款器點揀？**  
A：要簡單「一人一 EOA、歸集用普通 `transfer`」→ HD。要統一合約邏輯（memo 模擬、自動 forward、白名單 hook）→ CREATE2 factory。成本：HD 管理海量私鑰／簽名；CREATE2 管理合約部署同升級風險。好多 CEX 原生幣／ETH 用 HD，合約代幣收款亦可能用 HD EOA 收 ERC-20。

**Q：seed 洩漏點算？同「只洩漏一條用戶充值私鑰」？**  
A：seed／master 洩漏 ≈ 整棵樹可被掃空 → 最高級事故，要旋轉帳戶、遷移資金。單條 leaf 私鑰洩漏：攻擊者可清空該地址；若用咗非 hardened 不當設計， theoretically 可能擴大。運維：最小權限、地址監控、異常出金告警、保險庫分層。

## 小練習

**題：** 產品要為每個用戶生成 ETH 充值地址。方案 A：熱服持有 `m/44'/60'/0'` 嘅私鑰，按 `address_index=userId` 派生並本地簽名歸集。方案 B：熱服只持 `xpub`，歸集請求打去 HSM。你會點評風險？入帳掃描兩邊有冇分別？

**參考答案：**  
- **入帳掃描**：兩邊一樣——只需地址列表／`xpub` 派生地址 + 掃鏈；唔需要私鑰。  
- **方案 A 風險**：熱服被攻＝該 account 下已派生（同未來可派生）地址資金可被轉走；`userId` 當 index 要防碰撞／重用；seed 或 account key 喺熱服係反模式。  
- **方案 B**：熱服只能生成地址同發起「待簽」任務；簽名喺 HSM／MPC，可加策略（額度、白名單、多人審批）。代價係延遲同工程複雜度。  
- 面試加分：講清 **path 穩定性**、備份／恢復演練、同 CREATE2 方案共存時「地址→user」映射仍係唯一真相。
