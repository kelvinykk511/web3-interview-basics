# L2 出金真實成本：L1 data fee + blob 費波動

## 人話

喺 L2（Arbitrum、Base/OP）出一筆金，你俾嘅 gas 其實分兩份：**L2 執行費**（好平）+ 把呢筆交易數據貼返去 L1 嘅費用（**L1 data fee**）。後者先係大頭，而且跟 L1 blob 市場浮動，同 L2 本身幾忙冇必然關係。

## 面試短答

L2 tx 成本 = L2 execution gas × L2 gas price + L1 data fee。OP Stack（Base/OP）嘅 L1 fee 獨立計，可以用預部署合約 **GasPriceOracle**（`0x420000000000000000000000000000000000000F`）`getL1Fee(txBytes)` 估，上鏈後 receipt 有 `l1Fee` 欄位；Arbitrum 就將 L1 成本折算成額外 gas 加入 `gasUsed`，可用 **NodeInterface** `gasEstimateL1Component` 拆出嚟。EIP-4844 後 L2 主要用 blob 貼數據，blob base fee 有自己一套 1559 式調價，blob 需求一爆（例如 blob 用量長期滿過 target）就會指數式上升。所以 CEX 嘅 fee oracle 要分開追 **L2 gas** 同 **L1 data/blob** 兩條，唔可以淨係睇 `eth_gasPrice`。

## 常見追問（連答案）

**Q1：點解 L2 出金費有時突然貴幾倍但 L2 唔塞？**  
A：因為 L1 data fee 跟 L1 base fee / blob base fee 走，L1 或 blob 市場貴咗，L2 tx 就貴，同 L2 擠唔擠無關。

**Q2：`eth_estimateGas` 喺 OP Stack 夠唔夠？**  
A：唔夠，佢只估 L2 execution gas；要另外加 `getL1Fee`。Arbitrum 嘅 `estimateGas` 已經包埋 L1 部分（所以 gas limit 數字睇落好大），兩條鏈嘅估法唔同，錢包服務要按鏈寫 adapter。

**Q3：點樣減 L1 data fee？**  
A：交易數據越細越平：用 ERC-20 transfer 唔好包多餘 calldata、批量出金（一筆 multicall/批量合約攤分固定開銷）、避免喺 blob 價高峰做非緊急嘅歸集/再平衡（歸集可以排隊等平價時段）。

**Q4：帳本點記？**  
A：實際 gas 成本 = L2 部分 + `l1Fee`（OP）或 `gasUsed × effectiveGasPrice`（Arbitrum 已含），以 **receipt 為準**入成本帳，唔好用估算值入帳。

## 小練習

**題：** Base 上出 USDT，估算時只用 `eth_estimateGas × gasPrice` 定咗 gas 預算同用戶手續費，blob 價暴升嗰日熱錢包 ETH 消耗快過預期、用戶費唔夠蓋成本。問題喺邊？點改？

**參考答案：** 漏咗 L1 data fee，而佢正正係跟 blob 市場升嘅大頭。改：(1) 估算 = L2 execution + `GasPriceOracle.getL1Fee(簽好嘅 tx bytes)`；(2) fee oracle 分開監控 L2 gas 同 L1/blob 價，設上限告警；(3) 熱錢包 ETH gas 水位按「包含 L1 fee 嘅實際成本」計 burn rate；(4) 成本帳用 receipt `l1Fee`；(5) 非緊急歸集排隊避峰。
