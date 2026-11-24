# Ethereum Fast Confirmation Rule（FCR）：`safe` 標籤同 CEX 充值確認

> 銜接：你已經識 [充值 reorg／確認數／最終性](./deposit-reorg-confirmation-depth.md)、[L2 充值最終性](./l2-deposit-finality.md)、同 [多 RPC failover](./rpc-multi-node-failover.md)。FCR 唔改最終性，亦唔係硬分叉；佢改嘅係共識客戶端點樣計「呢塊夠唔夠安全」，再把結果接到現有 JSON-RPC 嘅 `safe` 標籤。對 CEX 嚟講，最大陷阱係：**你本來用緊嘅 `safe`，啟用 FCR 之後語義會變快好多，而且唔同 RPC／節點可以唔一樣。**

官方（以 2026-11-24 文件為準）：[EF Protocol Consensus：FCR](https://consensus.ethereum.foundation/themes/fast-confirmation)、[Consensus Specs：Fast Confirmation](https://ethereum.github.io/consensus-specs/specs/phase0/fast-confirmation/)、[fastconfirm.it](https://fastconfirm.it/)。Gnosis Bridge 官方寫由 **2026-10-08** 起喺生產用 FCR 處理由以太坊出發嘅橋接，係生態入面第一個公開生產案例（[Gnosis 文件](https://docs.gnosischain.com/bridges/fast-confirmation-rule)）。Alchemy／Chainstack 等 RPC 文件亦寫明：FCR 多數要 **顯式開啟**，唔好假設主網公共 `safe` 已經係 FCR。

## 人話（從你熟的 CEX）

以太坊入帳一向有三層速度，唔好混為同一種「確認」：

| 訊號 | 大約要幾耐 | 保證 | 點查 |
| --- | --- | --- | --- |
| `latest`／k-deep（例如等 12 塊） | 即時，或 k×12 秒 | 啟發式，冇形式化安全保證 | `latest` 或自己數塊 |
| **FCR（開啟後嘅 `safe`）** | 通常 1 個 slot，約 12–13 秒；有時 1–2 個 slot（12–24 秒） | 假設成立時，呢塊會進入 canonical 並最終被 finalize | `eth_getBlockByNumber("safe")` |
| 最終性 `finalized` | 通常約 13–19 分鐘（約 2 個 epoch） | 要推翻至少 1/3 質押被 slash；唔依賴網絡同步假設 | `finalized` |

FCR 數嘅係 **驗證者 attestation 權重**，唔係「再等幾塊就當穩」。官方兩條假設：

1. **同步性：** 誠實驗證者喺某個 slot 發出嘅 attestation，會喺 **同一個 slot 結束前** 被所有誠實驗證者收到（EF 寫成網絡延遲少於約 8 秒）。
2. **誠實多數：** 至少 **75%** 質押誠實並參與（Byzantine 上限 25%）。

兩條都成立，fast-confirmed 嘅塊保證會被 finalize。假設唔成立時，最常見係 **活性失敗**：FCR 唔再向前、確認變慢，回退到最終性，錢唔會憑空消失。極少數情況先會有 **安全性失敗**：已經 fast-confirm 嘅塊被 reorg，而且 **唔使有人被 slash**。官方講清楚：FCR **冇經濟安全**（違規冇質押可燒），所以佢唔係最終性嘅替代品。fastconfirm.it 亦寫：對手只要控制超過約 5% 質押，就可以拖慢確認（例如約 15% 再多等一個 slot，約 20% 再多等三個 slot）。

對交易所最關鍵嘅工程事實：**唔使新 RPC 方法，但會改 `safe` 嘅意思。** 未開 FCR 時，`safe` 通常係最新 **justified** 塊，大概落後 `latest` 一個 epoch（約 32 塊、6–13 分鐘）。開啟之後，`safe` 變成最新 fast-confirmed 塊，通常只落後幾塊。`finalized` 不變。所以如果你哋入帳條件早已寫「等到 `safe`」，運維喺共識客戶端加旗標、或者供應商默默開啟 FCR，等價於 **冇經過產品／風控會議就縮短確認政策**。fastconfirm.it 示例旗標：`lighthouse bn --enable-fast-confirmation`。Alchemy 寫 FCR **預設關閉**，要申請先改你條 endpoint 嘅 `safe`；Chainstack 文件（查閱當日）寫佢哋嘅 Hoodi Global Nodes 已跟 FCR，但 **Mainnet／Sepolia Global Nodes 嘅 `safe` 仍係 justified**。判別方法好實用：對同一 endpoint 比較 `latest`／`safe`／`finalized`——FCR 正常時 `safe` 只落後幾塊；未開啟時 `safe` 大約比 `finalized` 快一個 epoch；FCR 回退時 `safe` 可以同 `finalized` 重合，之後又可能一次過跳前好多塊。跟 `safe` 一定要記 **塊 hash**，唔好假設塊號單調向前。

實務建議按金額同可否回滾分層：大額、出金已廣播、金庫結算、不可逆操作，繼續等 `finalized`。零售 ETH 充值若要用 FCR 換速度，必須係 **顯式政策**：自己跑（或確認供應商已開）FCR 節點、客戶端多樣化、硬分叉／網絡不穩時回退最終性，並監控「已 confirm 嘅 hash 被換走」或「finalize 結果同之前 confirm 唔一致」。Gnosis Bridge 就係呢套：自建節點、唔靠單一公共 RPC、客戶端唔過半、唔穩就回退最終性。`latest` 仍然唔能入可用。k-deep 比 FCR 更弱、亦不一定更快；唔好以為「等 12 個確認」就自動等同或強過 FCR。

## 面試短答

1. **FCR 唔係硬分叉，亦唔等於最終性。** 共識客戶端功能；假設（同步 + ＜25% 作惡）成立時，約一個 slot 內確認一塊會被 finalize 嘅塊。
2. **開啟後，現有 `safe` 標籤改指最新 fast-confirmed 塊。** `finalized` 不變。唔好假設每條主網 RPC 都已開啟。
3. **保證弱過最終性。** 冇 slash 墊底；條件轉差通常回退變慢，極端先可能 confirm 後 reorg。
4. **比 k-deep 更有根據，但政策要顯式。** 已有系統若用 `safe` 入帳，開啟 FCR 等於悄悄加快入帳，要風控批准、hash 游標同回退預案。
5. **大額／不可逆繼續用 `finalized`。** 自建或核實過嘅節點、客戶端多樣化、升級期間回退。

## 常見追問（連答案）

**Q1：`safe` 而家係咪全部等於 FCR？可以信公共 RPC 嘅 `safe` 入帳嗎？**  
A：唔等於。`safe` 視乎嗰個節點／供應商有冇開 FCR。同一標籤，可能係 justified（數分鐘），亦可能係 fast-confirmed（十數秒），回退時甚至同 `finalized` 一樣。公共 RPC 唔標明就唔好當已開。要開，就自己開共識客戶端旗標，或者向供應商書面確認邊條 endpoint 已開。入帳政策寫明用邊種訊號，唔好只寫一個 `safe`。

**Q2：FCR 確認咗係咪就唔會 reorg？同最終性差喺邊？**  
A：喺官方兩條假設之下，確認咗嘅塊會進入 canonical 並被 finalize。最終性係另一種更強保證：要推翻要燒至少 1/3 質押，而且唔依賴「訊息一定喺 slot 內送到」。FCR 違規冇質押可罰。條件轉差時多數係暫時確認唔到、回退最終性；極端先會出現已 confirm 塊被換走。所以 FCR 係「比數塊更嚴謹嘅快速訊號」，唔係「13 秒版嘅 finalized」。

**Q3：零售充值用 FCR、大額用最終性，會唔會前後矛盾？出金呢？**  
A：唔矛盾，重點係分層同寫明假設。零售入可用可以接受「假設成立下約 13 秒、極端要有 reorg runbook」；金庫調撥、已向用戶廣播嘅出金、橋嘅放行，應繼續等 `finalized`。一個實務：內部狀態分成「FCR 已見／可展示」同「finalized 先入可用／先撥出」。Chainstack 甚至建議入帳同放行都用 `finalized`，把 FCR `safe` 留給介面展示同已處理 reorg 嘅索引——呢個係偏保守嘅供應商建議，面試講得出取捨就加分。

## 小練習（題目 + 參考答案）

**題目：** 組員打算：(1) 所有 ETH 充值改為 `eth_getBlockByNumber("safe")` 即入可用，因為「而家 `safe` 得十數秒」；(2) 繼續用現有公共主網 RPC，唔改客戶端旗標；(3) 大額出金同金庫結算都跟同一個 `safe`；(4) 游標只存塊號，認為 `safe` 只會向前。有咩問題？點改？

**參考答案：**

- **先核實呢條 RPC 嘅 `safe` 到底係 FCR 定 justified。** 對比 `latest`／`safe`／`finalized` 落後幾多塊；未開啟就仍然係數分鐘級 justified，唔係十數秒。要 FCR 就顯式開啟（例如 Lighthouse `--enable-fast-confirmation`）或換已確認嘅 endpoint，唔好猜公共 `safe`。
- **入帳政策分層，唔好所有資金跟同一個 `safe`。** 大額、出金放行、金庫結算用 `finalized`。零售若用 FCR，要書面假設（＜25% 作惡、slot 內收到 attestation），並有「已 confirm hash 被換走／finalize 結果唔一致」嘅告警同回滾 runbook。
- **游標用塊 hash。** FCR 回退時 `safe` 可以停住、一次過跳前、甚至退回 finalized；只記塊號會漏掃或重複。`latest` 仍然唔入可用。硬分叉或網絡不穩時預先切回最終性。
