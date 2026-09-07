# ERC-20 Transfer 事件（CEX 充幣掃描）

## 人話

原生幣（ETH）轉帳睇 transaction 嘅 `to` + `value`。  
代幣（USDT／USDC）唔係「value 欄有數」——係打去**代幣合約**，合約 emit `Transfer` 事件；CEX 掃 logs 先知邊個地址收到幾多。

用後端類比：原生幣 ≈ 直接改帳戶餘額；ERC-20 ≈ 某個「代幣服務」發咗一條「A→B 轉咗 X」嘅事件，你要訂閱／掃呢條事件先入帳。

## 面試短答

- ERC-20 標準事件：`Transfer(from, to, value)`（indexed：from、to）
- 充幣掃描：按區塊掃 `eth_getLogs`（合約地址 + Transfer topic + to=充值地址）或訂閱 logs
- 入帳鍵：`txHash` + `logIndex`（同一 tx 可有多筆 Transfer）+ 合約地址（幣種）+ 金額 + `to`
- 陷阱：假合約同名／錯 chain、fee-on-transfer、特殊代幣（冇標準 Transfer）

## 常見追問＋答案

Q：點解只睇 tx.to == 用戶充值地址會漏 USDT？  
A：用戶轉 USDT 時 tx.to 係 **USDT 合約**，真正收款地址喺 log 嘅 `to`。只 match 原生轉帳會漏晒代幣充值。

Q：同一筆 tx 點解要記 logIndex？  
A：一筆交易可以觸發多個 Transfer（例如路由／聚合）。冪等鍵要用 `txHash + logIndex`（再加 chainId／合約），唔好淨用 txHash。

Q：內部轉帳（internal tx）同 Transfer 有咩分別？  
A：合約 call 導致嘅原生幣移動未必出現喺頂層 tx.value；代幣則靠 event。索引要分清楚「原生／代幣／內部」，CEX 產品多數以標準 Transfer + 原生轉帳為主。
