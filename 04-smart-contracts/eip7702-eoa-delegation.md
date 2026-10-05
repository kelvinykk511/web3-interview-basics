# EIP-7702：EOA 委託代碼（點樣打破「有 code＝合約」）

## 人話（從你熟的 CEX）

以前 CEX 風控有條好簡單嘅規則：`eth_getCode(addr) == 0x` ⇒ 普通錢包（EOA），有 code ⇒ 合約。提現白名單、簽名綁定（ecrecover vs EIP-1271）、「禁止出金去合約」全部靠呢條。

Pectra 升級（2025）加咗 **EIP-7702**：一個 EOA 可以簽一張「授權」，指定一個合約做佢嘅「代碼實現」。之後呢個地址：

- **私鑰仍然有效**（佢仍然係 EOA，可以照常簽 tx）
- 但 `eth_getCode` 會返 23 bytes：`0xef0100 ‖ 委託合約地址`（delegation designator）
- 任何人 call／轉 ETH 去呢個地址，會**執行委託合約嘅代碼**

類比：一個普通用戶帳戶，突然綁咗一個「自動化腳本」——仍然係同一個人用同一把鑰匙登入，但入帳時會觸發腳本。

## 面試短答

- 新 tx type `0x04`，帶 `authorization_list`；每條授權 = `(chainId, delegateAddress, nonce, 簽名)`，由 EOA 本人簽，但**可以由任何人打包送上鏈**（例如 relayer 代付 gas）。
- 授權處理時 EOA 嘅 **nonce +1**；`chainId = 0` 表示**所有鏈通用**（風險位）。
- 撤銷：再簽一張授權指向 `address(0)`，清走 designator；但**私鑰永遠有最高權限**，委託唔會令私鑰失效。
- 對 CEX 影響三點：
  1. **地址分類**：`getCode` 非空唔再等於「純合約」——要識別 `0xef0100` 前綴，當「有委託嘅 EOA」獨立一類。
  2. **出金**：轉 native ETH 去 7702 地址會跑對方代碼；寫死 `gas=21000` 可能失敗，或者對方 receive 再轉走（入帳變 internal transfer）。
  3. **簽名驗證**：有 code 唔代表唔可以 ecrecover；同一地址可能兩種驗證都成立（私鑰簽 + 1271）。

## 常見追問（連答案）

**Q：提現風控「禁止合約地址」要點改？**  
A：唔好再用「code 是否為空」一刀切。先讀 code：空 → EOA；以 `0xef0100` 開頭且長度 23 → 7702 委託 EOA，解析出 delegate 地址，對照白名單（例如知名錢包嘅 delegate 實現）／黑名單（已知 drainer）；其他 → 真合約，走原有合約政策。出金 gas 用 `eth_estimateGas` 而唔係寫死 21000。

**Q：用戶充值地址（CEX 自己生成嘅 HD 地址）會唔會中招？**  
A：CEX 自己唔簽授權就唔會有委託。但要防兩件事：(1) 歸集／熱錢包簽名服務**絕對唔可以**簽任何 type-4 授權（簽名機要白名單 tx type）；(2) 監控自家地址 `getCode` 變成非空 = 安全事件（代表有人用私鑰簽咗授權，等同私鑰洩漏或簽名服務被濫用）。

**Q：點解 7702 會影響熱錢包 nonce 管理？**  
A：授權被處理時 authority 嘅 nonce 會 +1，而且**第三方可以打包你嘅授權**。如果熱錢包本身用 7702（例如批量出金合約），nonce 唔再只由你自己發 tx 推進——nonce allocator 必須以鏈上 `pending/latest` 對帳，唔可以純靠本地計數。

**Q：用戶層面最大嘅騙局係乜？**  
A：釣魚網站騙用戶簽一張授權（尤其 `chainId=0`），delegate 指向 drainer 合約——之後所有入到呢個地址嘅資產（包括從 CEX 提現過去嘅）會被自動掃走，即「sweeper」。CEX 客服見到「提現到帳即被轉走」，可以查 `getCode` 有無 `0xef0100` 前綴做第一步 triage。

## 小練習

**題：** 提現服務收到三單，`eth_getCode(to)` 分別返：(a) `0x`；(b) `0xef0100` + 20 bytes，delegate 係已知 drainer；(c) 一大段 runtime bytecode。另外，每日巡檢發現一個 CEX 自家充值地址 `getCode` 由 `0x` 變成 `0xef0100…`。每單點處理？

**參考答案：**  
(a) 普通 EOA，照常走 EIP-55／黑名單／限額流程。(b) 7702 委託 EOA 且 delegate 屬 drainer → 攔截，轉風控人工審核同提醒用戶（大機會已被騙簽授權），唔好直接出金。(c) 真合約 → 按原有合約地址政策（拒絕或白名單合約），出金 gas 用估算值。自家充值地址出現 designator → 當私鑰／簽名服務洩漏處理：即刻停該地址入帳歸集流程、凍結相關簽名 key、盡快把資產歸集到安全地址（要計算 delegate 代碼會唔會攔截轉出），並開安全事件單。
