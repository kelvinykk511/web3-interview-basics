# 歷史過期（partial history expiry）：CEX 歸檔、對帳同索引

> 銜接：你已經識節點／RPC 故障切換、充值掃鏈同 event log 陷阱。今日講嘅係 **合併前嘅 block body 同 receipt 已經可以從普通執行節點消失**，同 PeerDAS／blob 約 18 日窗口、以及尚未上路嘅完整 EIP-4444 滾動過期，都唔係同一件事。

官方（以 2026-11-28 文件為準）：[EF Blog 2025-07-08 Partial history expiry](https://blog.ethereum.org/2025/07/08/partial-history-exp)、[ethereum.org：Statelessness / history expiry（頁面更新 2026-11-16）](https://ethereum.org/roadmap/statelessness/)、[EIP-4444](https://eips.ethereum.org/EIPS/eip-4444)、[社群歷史存檔目錄](https://eth-clients.github.io/history-endpoints/)。2025-07-08 起，各執行層客戶端都**支援**修剪合併前歷史；完整、按窗口滾動嘅 EIP-4444（規格寫 `HISTORY_PRUNE_EPOCHS = 14,299`，約兩個月）截至 ethereum.org 2026-11-16 仍在討論，**未**作為全網強制規則上線。

## 人話（從你熟的 CEX）

以前幾乎每台「全節點」都默默存住由 2015 創世到而家嘅交易包同回執，磁碟輕易過 TB。但其實驗證新塊、跟住鏈頭，唔再需要呢啲舊 body／receipt——PoS 之後大家早已用弱主觀檢查點同步，而唔係由創世一筆筆重放。

2025 年 7 月嘅 **partial history expiry** 做咗第一刀：執行客戶端可以丟掉 **The Merge 之前**（主網 block **< 15537394**，即至 15537393）嘅 **block body 同 receipt**，官方估算大約可省 **300–500 GB**，令節點較容易放進 2 TB 碟。**Header 會留住**，所以仍然可以由創世用加密方式核對鏈條，唔係「歷史被抹掉」。被修剪嘅正文同回執，改由 archive、torrent、`era1`／`ere` 檔、以及 Portal Network 呢類協議外來源提供。

用內部系統比喻：撮合核心只保留近期成交同指數，2017 年嘅逐筆成交改放冷庫。核心仍然可以對「當日盤」負責，但你若把審計、稅務、舊充值補掃、合規調證直接打去一台剛同步好嘅交易核心，佢可能答你「呢段已經唔在線」。

幾個容易踩嘅坑：

- **唔係所有客戶端都已經自動清空舊庫。** EF 公告寫：Nethermind 由 1.32.2 起，**新同步**預設不再下載合併前 body／receipt（可用 barrier=0 改返）；已有數據要另開修剪。Geth／Besu 等要顯式 flag 或離線修剪命令。唔好假設「我跑 geth 全節點，所以一定有 2018 年 receipt」。
- **`eth_getLogs`／舊塊查詢跨入已修剪高度時，應回錯誤，而唔係靜默少結果。** Execution API 定義錯誤碼 **4444**，訊息 `Pruned history unavailable`。若業務把錯誤當成「呢段冇 Transfer」，就會漏掃舊充值或者錯誤結案。
- **修剪嘅係歷史 body／receipt，唔係 state archive，亦唔係 blob。** 帳戶／合約存儲點樣保留係另一套 archive 模式。Blob sidecar 約 18 日窗口由 4844 就有，同合併前 receipt 無關。完整 EIP-4444 滾動窗口（約兩個月）尚未全網強制，唔好提早當「所有 3 個月前嘅 log 都會消失」。
- **RPC 供應商同自建節點策略已經分流。** 有啲 endpoint 為慳碟已經冇合併前歷史。CEX 要用兩條路：跟鏈頭嘅熱節點，以及明確標示「完整歷史／archive」嘅來源，並且要能核對（header 仍在、存檔格式有 checksum／accumulator）。

對 CEX 最實際嘅影響：

- **充值補掃、舊 tx 查詢、對帳、稅務、法律調證、某些 PoR／審計抽樣** 唔可以只打普通全節點。
- **監控要把 4444／pruned history 當成獨立故障**，同「節點落後、RPC 超時、真係冇 log」分開。
- **新部署節點先核對客戶端預設。** 尤其係新同步嘅 Nethermind，一開箱就可能冇合併前歷史。
- **自己做長期存檔** 可以跟社群 `era1`（合併前）同 `ere`（完整執行層歷史）鏡像，而唔係只靠單一商業 RPC。

## 面試短答

1. **2025-07-08 起，執行客戶端支援 partial history expiry：** 可刪合併前（主網 block < 15537394）嘅 body 同 receipt，大約省 300–500 GB；header 保留，方便由創世核對。
2. **完整 EIP-4444 滾動過期尚未全網強制。** ethereum.org 於 2026-11-16 仍寫「under discussion」。唔好把而家講成「所有節點只留最近兩個月歷史」。
3. **對 CEX：舊日誌同舊回執要有專門 archive 來源。** 熱節點負責鏈頭；補掃、審計、調證走 archive／`era1`／`ere`／可靠歷史供應商。
4. **跨已修剪區間嘅 `eth_getLogs` 應報錯（例如 4444），唔可當成空結果。** 靜默少 log 比報錯更危險。
5. **唔好同 state expiry、blob 過期、PeerDAS 混為一談。** 各改各嘅數據面。

## 常見追問（連答案）

**Q1：點解丟得掉合併前嘅 body／receipt，仲算唔算「全節點」？鏈係咪仍然可由創世驗證？**  
A：驗證鏈頭已經唔依賴重放 2016 年每筆交易。Header 仍在，可以核對歷史哈希鏈；body／receipt 改由專門存檔提供。日常共識同出塊唔受影響。所謂「全節點」而家更常指「跟到頭、有近期歷史同當前 state」，唔再自動等於「由創世起每筆 receipt 都在本地」。若面試要精確，應講你嗰台係 pruned、full 定 archive，以及歷史由邊個高度開始。

**Q2：我們用自建節點做一次 2017 年到而家嘅 ERC-20 充值補掃，`eth_getLogs` 回空陣列，係咪代表鏈上真係冇呢啲入帳？**  
A：先查係唔係打中已修剪區間。正確行為係對已修剪高度回 4444／pruned-history 錯誤。若某驅動或中間件把錯誤吞成 `[]`，補掃會漏帳。應改打 archive，確認`fromBlock` 落喺該來源仍然保留嘅區間，並為 4444 單獨告警。空陣列只喺「來源保證覆蓋呢段、而且成功返回」時先能當成真冇事件。

**Q3：Nethermind、Geth 預設一樣嗎？我已經跑咗兩年嘅舊節點會唔會某日自己消失歷史？**  
A：唔一樣。EF 公告：Nethermind 1.32.2 起**新同步**預設跳過合併前 body／receipt；已有數據唔會無聲自動清掉，要另開修剪（其文件中 `History.Pruning` 預設 `Disabled`，清舊庫係 opt-in）。Geth 新節點要用 `--history.chain postmerge` 先會跳過；舊庫要用 `geth prune-history` 等顯式步驟。結論：盤點每台節點嘅客戶端、版本、啟動參數同實際最早可服務塊高，唔好用「全節點」三個字代替配置。

**Q4：同 EIP-4444 寫嘅 14,299 epoch、同 blob 18 日、同 state expiry 有咩分別？**  
A：而家已落地嘅係「合併前 body／receipt 可修剪」。EIP-4444 還提出之後滾動刪超過 14,299 epoch（約 63 日）嘅歷史，呢一步尚未全網強制。Blob sidecar 約 18 日請求窗口係 4844／共識層熱數據，對象係 blob 而唔係 2018 年嘅 ETH transfer receipt。State expiry 講緊帳戶狀態，更未係今日呢刀。規劃存檔時三條時間軸要分開。

**Q5：合併後（例如 2024 年）嘅 log 而家會唔會被普通節點丟掉？資料去咗邊？點核對真偽？**  
A：按而家已經上船嘅規則，合併後 body／receipt 仍預期留喺未開滾動修剪嘅普通全節點。長期方案同社群存檔係 `ere`（完整執行層，覆蓋合併前後）、合併前另有 `era1`，共識層有 `era`；亦有 torrent 同 Portal Network。核對可以靠仍然保留嘅 header，以及存檔自帶嘅 checksum／epoch accumulator，唔好只信單一 HTTP 鏡像。商業 RPC 要喺合約／runbook 寫明歷史保留範圍。

**Q6：充值即時入帳流程要改嗎？熱錢包出金呢？**  
A：跟鏈頭、確認數、finalize 之後入帳嘅主路徑通常唔使改，因為新充值唔會落喺合併前。要改嘅係：舊單補掃、用戶出示舊 tx hash 嘅客服查詢、審計抽樣、合規調證、同任何「由 genesis 掃到而家」嘅對帳工作。出金廣播只關心 mempool 同近期狀態，但若你用同一條 RPC 做歷史查詢，就要分開 endpoint，以免被修剪策略連累。

## 小練習（題目 + 參考答案）

**題目：** 運維新開一台 Nethermind（預設、新同步）同埋一台多年前就在跑、未做過 prune 嘅 Geth，都叫做「全節點」，並把以下工作打去隨機一台上：(1) 即時 ETH／ERC-20 充值掃鏈（只掃最近數百塊）；(2) 補掃 2020 年某代幣合約嘅 Transfer，用來對舊客訴；(3) 審計要抽 2019 年一筆出金 receipt；(4) 監控見到 `eth_getLogs` 回空陣列就當「呢段冇事件」；(5) 有人主張「既然 EIP-4444 都寫咗，2024 年嘅 log 我們亦只保證留兩個月」。有咩問題？點改？

**參考答案：**

- **即時近頭掃鏈可以走熱節點**，兩台通常都應付到，因為唔依賴合併前 receipt。
- **2020／2019 嘅 log 同 receipt 必須走仍保留合併前歷史嘅來源。** 新同步 Nethermind 預設可能根本冇呢段；隨機負載均衡會令客訴同審計間歇失敗。
- **空陣列 ≠ 冇事件。** 已修剪區間應出現 4444／pruned-history 錯誤；要當故障或走錯來源，並補掃 archive。若見到空結果，先核對該來源嘅最早可服務塊高。
- **唔好提早執行完整 EIP-4444 窗口。** 滾動約兩個月歷史尚未全網強制；2024 年 log 仍應可喺普通未滾動修剪嘅全節點或 `ere` 存檔取得。產品／法務承諾要按實際保留策略寫，而唔係按一張未上線規格提前縮水。
- **把熱路徑同歷史路徑分開，** 盤點每台節點最早塊高，並自備或簽約一套可核對嘅歷史存檔。
