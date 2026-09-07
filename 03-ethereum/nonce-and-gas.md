# Nonce 與 Gas（CEX 出金常問）

## 人話

**Nonce**：呢個地址發出嘅第幾筆交易（由 0 起計）。要按序；中間缺一個，後面會卡住。  
**Gas**：付費請網絡執行／打包。Gas limit = 預算上限；gas price／base fee+tip = 願意出幾貴。

## 面試短答

- 熱錢包連續出金：要管 nonce 分配，避免並發搶同一 nonce
- 交易 pending 太耐：可提價 replace（同 nonce 更高費）或視情況 cancel
- Gas 估錯：limit 太低會失敗；太高通常只係上限，實際按用量扣（視鏈）

## 常見追問＋答案

Q：點解出金「卡住」後面全部唔出？  
A：常見係某 nonce 交易卡喺 mempool，後面更高 nonce 要等佢先；要替換／加速嗰筆，或等佢掉。

Q：同一 nonce 廣播兩筆會點？  
A：網絡只會接受一筆上鏈；另一筆被替代／丟棄。CEX 要有清晰嘅 signer 序列化或 nonce 鎖。
