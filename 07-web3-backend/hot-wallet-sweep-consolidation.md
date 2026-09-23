# 熱錢包 Sweep／歸集（HD 充值後 consolidation）

## 人話（從你熟的 CEX）

用戶充值地址（HD 派生或 CREATE2）收到幣之後，錢散落喺成千上萬個子地址。出金熱錢包／運營池要集中流動性，就要 **Sweep（歸集）**：定期或閾值觸發，把子地址餘額轉去歸集地址（熱運營或冷錢包中轉）。

人話：似支付系統「商戶子賬戶 → 總清算戶」；鏈上每筆 sweep 都係真 tx（nonce、gas、ERC-20 `transfer`／ETH 轉賬），仲要處理「地址有 token 但冇原生幣付 gas」呢個經典坑。

## 面試短答

- **目標**：降低分散地址運維成本、滿足熱錢包流動性、方便對冷錢包上存；唔係「一充值即刻轉走」都得——要權衡 gas、確認數、風控。
- **觸發**：餘額 ≥ 閾值、地址數／總額告警、定時 batch、冷熱水位低於水位線時加速歸集。
- **路徑**：子地址 → 歸集熱地址 →（可選）冷錢包；ETH／原生幣 sweep 同 ERC-20 sweep 流程不同（ERC-20 要子地址有 gas 或用 meta-tx／relayer／4337 paymaster）。
- **安全**：sweep 用專門 signer 權限、金額同地址白名單、速率限制；私鑰仍按 HD／HSM 模型，業務庫只存 xpub／地址索引。
- **對帳**：每筆 sweep 有 `fromAddr + txHash` 冪等；帳本從「用戶充值已入帳」唔因 sweep 再入一次——sweep 係 **custody 內部調撥**，唔改用戶餘額。

## 常見追問（連答案）

**Q：用戶充值入帳咗，sweep 失敗會唔會影響用戶餘額？**  
A：唔應該。入帳喺掃到充值 tx 並達確認數時已經 credit 用戶；sweep 係公司地址之間搬倉。失敗只影響熱錢包流動性同 gas 成本，要用獨立狀態機（`PENDING_SWEEP`／`SWEEP_FAILED`）重試，**禁止**因此扣用戶或重複 credit。

**Q：ERC-20 子地址冇 ETH，點付 gas？**  
A：常見幾種：（1）先由 gas 贊助地址打一小筆原生幣再 sweep token；（2）歸集合約／permit 類設計（視幣種）；（3）4337 + paymaster（較新）；（4）有啲鏈用 fee-token。CEX 面試常答「gas tank／贊助 tx + 批量補油」，並強調補油本身都要防濫用同對帳。

**Q：HD 同 CREATE2 歸集有咩差別？**  
A：HD EOA：每地址自己 nonce，要串行或按地址並行隊列簽名 `transfer`。CREATE2 工廠合約：可能 `sweep(token, salts…)` 一次收多個，或用戶直接打進可執行邏輯嘅合約。後者 gas 結構同權限模型不同，但對帳仍係內部調撥。

**Q：點解唔可以「見充值就即 sweep」？**  
A：確認數未夠就轉走，reorg 可能令充值回滾但 sweep 已出（或相反），對帳極痛；細額歸集會被 gas 吃光；仲增加熱路徑簽名頻率同攻擊窗。實務：達確認／safe 後入候選池，按閾值同 gas price 窗口批量 sweep。

## 小練習

**題：** 十萬個 HD 充值地址，其中 2% 有 USDT 餘額，多數子地址 ETH=0。設計一個日歸集任務：點揀地址、點補 gas、點保證唔雙花 nonce、點同用戶入帳流水分開？

**參考答案：**  
1. **候選**：掃索引庫 `deposit_addr` 餘額快照（或鏈上 `balanceOf` 批量／Multicall）≥ 閾值且充值已 `credited`（確認數達標）。  
2. **補油**：對 ETH=0 且要轉 ERC-20 嘅地址，由 gas tank 按 nonce 隊列打固定額；補油 tx 回執成功先入隊 sweep。  
3. **簽名／nonce**：每地址獨立 nonce 隊列（或鎖）；同一地址唔並行兩筆未確認 sweep；RBF 只同 nonce。  
4. **帳務**：sweep 流水類型=`INTERNAL_SWEEP`，關聯 `from=depositIdx → to=treasury`，**唔**寫用戶 ledger；監控歸集後 treasury 餘額同熱錢包水位。失敗重試唔產生用戶流水。
