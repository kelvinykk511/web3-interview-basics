# Polygon Agglayer v1.0：Unified Bridge 對 CEX 金庫／上架／充提

> 銜接：你已經識 [CCTP vs 橋接 USDC](./usdc-cctp-cross-chain-rebalancing.md)、[CCIP 2.0 金庫跨鏈](./chainlink-ccip-cex-treasury.md)、[LayerZero OFT vs CCTP](./layerzero-oft-vs-cctp-rebalancing.md)、[Circle Gateway 統一 USDC 餘額](./circle-gateway-unified-usdc-balance.md)、同 [OP Superchain Interop](./op-superchain-interop-cex.md)。Agglayer 係 Polygon 嘅 **統一橋 + 悲觀證明（pessimistic proof）**：多條鏈共用一套 lock／burn → 證明 → claim 流程，唔係「一個餘額行晒所有鏈」，亦唔係 Superchain 嗰種原生訊息。

官方（以 2026-11-24 文件為準）：[Agglayer 概述](https://docs.polygon.technology/interoperability/agglayer)、[Supported Chains](https://docs.polygon.technology/interoperability/agglayer/supported-chains)、[Asset Bridging](https://docs.polygon.technology/interoperability/agglayer/core-concepts/unified-bridge/asset-bridging)、[Bridge Components](https://docs.polygon.technology/interoperability/agglayer/core-concepts/unified-bridge/bridge-components)、[Pessimistic Proof](https://docs.polygon.technology/interoperability/agglayer/core-concepts/pessimistic-proof/architecture)、[Environments](https://docs.polygon.technology/chain-development/cdk/integration-partners/environments)。v1.0 公告（[2026-11-23 官方博客](https://polygon.technology/blog/agglayer-v1)）：以太坊→已連接鏈典型低於 2 分鐘，已連接鏈→以太坊低於 15 分鐘（以前約 90 分鐘）；Katana 係第一條行快速結算路徑嘅鏈；以太坊最終性等待喺合資格路徑上改為可選；下一步先至整合 FCR（公告寫「now in testing」），**唔好把 v1.0 當成已用 FCR 入帳**。

官方 Supported Chains 主網表（查閱當日；rollup ID ≠ chainId）：

| 鏈 | chainId | Rollup／Network ID | 類型 | Gas |
| --- | --- | --- | --- | --- |
| Ethereum | 1 | 0 | L1 | ETH |
| Katana | 747474 | 20 | ALGateway | ETH |
| Pentagon Games | 3344 | 16 | Validium | PC |
| Silicon.network | 2355 | 10 | Validium | ETH |
| X Layer | 196 | 3 | PP（悲觀證明） | OKB |
| Ternoa | 752025 | 13 | zkEVM | CAPS |
| Wirex Pay | 31415 | 8 | zkEVM | ETH |

測試網：Bokuto（737373／37）、Lumia（2030232745／35）。類型唔一樣：Validium 數據可用性喺鏈下；X Layer 係 PP sovereign mode。上架要按類型做盡調，唔好當成同一種「Polygon L2」。**Polygon PoS（chainId 137）唔喺呢張 Agglayer 表入面**，唔好把舊 PoS 橋同 Unified Bridge 混用。

主網 `AgglayerManager`（舊名 `RollupManager`）喺以太坊：`0x5132a183e9f3cb7c848b0aac5ae0c4f0491b7ab2`。每條鏈各自部署 `PolygonZKEVMBridgeV2`（`bridgeAsset`／`claimAsset`／`bridgeMessage`／`claimMessage`）；具體橋地址用官方 Bridge Service 嘅 network information 查，唔好抄舊鏈或測試網地址。

## 人話（從你熟的 CEX）

把它當成一條 **跨多條鏈嘅官方統一橋**，而唔係 Circle Gateway 那種「一個 USDC 餘額即時可撥」。流程仍然係兩段：

1. 源鏈 `bridgeAsset`：原生／ERC-20 被鎖或者燒掉，交易寫入呢條鏈嘅 Local Exit Tree。呢步成功 **只代表源鏈已鎖／已燒**，目標鏈仲未有幣。
2. 等 Global Exit Root（GER，`hash(RollupExitRoot, MainnetExitRoot)`）包含呢筆葉，再喺目標鏈 `claimAsset`：用 Merkle proof 驗證、防重複 claim，然後轉原生幣、鑄 mapped／wrapped token，或者（首次橋入嘅外國 ERC-20）先部署一條新代幣合約再鑄。

所以「官方話資產保持同一身份、唔使 wrap」係產品層表述；工程上目標鏈經常係 **另一個合約地址**。CEX 資產 key 必須係 `(chainId, token)`，再用官方 mapping 對，唔好假設 USDC／USDT 跨鏈同址，更加唔好把 mapped token 當 Circle 原生 USDC 或官方原生 USDT 混倉。L2↔L2 同樣要對 GER 做 claim，唔係自動到帳。claim 係目標鏈上一筆獨立交易，通常要 relayer 或自己提交；源鏈成功 ≠ 用戶已收到。

v1.0 把證明同證書流水線加快（即時出 certificate、Succinct 實時出證明、即時最終性嘅鏈唔再額外等），官方宣傳嘅「2 分鐘／15 分鐘」係典型結算時間，**唔係協議 SLA**。合資格路徑可以選擇唔等以太坊最終性（公告話大約省 15 分鐘），呢個係運營風險選擇：大額金庫應預設繼續等最終性。悲觀證明嘅含義亦要講準：一條被攻破嘅鏈 **唔可以提取多過自己鎖入嘅存款**，損失半徑停喺呢條鏈嘅鎖倉，但鎖喺呢條鏈嘅資金仍然可以沒。唔等於「跨鏈調撥零風險」，更唔等於 CEX 內部有一個跨鏈統一可用餘額。

上架／出金仲有幾條易踩：每條鏈 gas token 可以唔同（X Layer 要備 OKB，唔係 ETH）；rollup ID（例如以太坊 0、X Layer 3、Katana 20）唔能同 chainId 混用；用戶直接轉去橋合約唔等於已充值到你嘅熱錢包；高可用升級官方寫計劃 2027 年初，唔好把公告當成節點／API 已有交易所級 SLA。同 CCTP／CCIP／OFT／Superchain Interop 分工：USDC 優先官方 CCTP；Agglayer 用喺呢個統一橋生態入面嘅鏈（X Layer、Katana 等）；唔好雙路徑重複入帳。

## 面試短答

1. **Unified Bridge 係 lock／burn + 證明 + `claimAsset`，唔係統一餘額。** 源鏈成功只係在途，入帳認目標鏈 claim 實收。
2. **v1.0（2026-11-23 公告）加快 ZK 結算，但 FCR 尚未作為已上線能力來用。** 官方典型時間：ETH→連接鏈 ＜2 分鐘，回流 ＜15 分鐘；Katana 先行快速路徑；最終性等待可選，大額應保守。
3. **資產 key 用 `(chainId, token)`，另備 rollup ID。** 目標鏈代幣地址可同源鏈唔同；PoS 137 唔屬於呢張表。
4. **悲觀證明只限制「一條鏈最多只能拿走自己鎖進去嘅錢」。** 鎖喺被攻破鏈嘅庫存仍會受損。
5. **Gas token 跟鏈走。** 例如 X Layer 出金要備 OKB。

## 常見追問（連答案）

**Q1：可唔可以當 Agglayer 係「一個 ETH／USDC 餘額，各條連接鏈通用」，好似 Gateway？**  
A：唔可以。Gateway 係 Circle 嘅統一餘額產品；Agglayer 仍然要源鏈鎖／燒、目標鏈 claim。帳本應分「源鏈已鎖／已燒（在途）→ GER／證明齊備（可 claim）→ 目標鏈已轉／已鑄」。未 claim 就加目標可用等於自己墊付。

**Q2：悲觀證明係咪等於對手足被攻破都唔會蝕錢？同 Validium／PP 有咩分別？**  
A：只保證嗰條鏈提唔走多過自己嘅存款，隔離嘅係爆炸半徑，唔係零損失。另外每條連接鏈類型唔同：Validium 執行有 ZK，但 DA 喺鏈下；X Layer 係 PP sovereign mode。上架同確認政策要按鏈類型定，唔好用「都喺 Agglayer 上」一筆帶過。

**Q3：用戶喺以太坊 `bridgeAsset` 到我哋喺 X Layer 嘅地址，或者金庫從 Katana 調回 L1，幾時可以入帳／當已到？**  
A：都只認目標鏈 `claimAsset` 成功（`status=1`）同實際收款地址／金額。X Layer gas 係 OKB，熱錢包要備氣。Katana 即使喺快速路徑，大額回流都應自行決定是否仍等以太坊 `finalized`；唔好用官方「低於 15 分鐘」當已到帳。FCR 要等 Agglayer 正式整合並寫進你哋政策之後先用。映射代幣同原生 USDC／USDT 分倉。

## 小練習（題目 + 參考答案）

**題目：** 組員打算：(1) 用戶以太坊 `bridgeAsset` 成功就喺 X Layer 預入可用；(2) 假設 USDC 合約地址同以太坊一樣；(3) 把 Polygon PoS（137）充值併入同一條 Agglayer 掃描器，network id 用 chainId；(4) X Layer 出金只備 ETH 做 gas；(5) 因為 v1.0 已上線，金庫回流唔再等最終性，並把 FCR `safe` 當到帳。有咩問題？點改？

**參考答案：**

- **只認目標鏈 `claimAsset` 實收。** 源鏈成功只記在途。X Layer 嘅 network ID 係 3，唔係 196；資產 key 用 `(196, 映射後 token)`。
- **用官方 token mapping，唔假設同址。** 首次橋入可以部署新 ERC-20。Mapped USDC ≠ Circle 原生 USDC，分倉、分掃。
- **PoS 137 唔喺 Agglayer 支持表，分開橋同掃描器。**
- **X Layer 熱錢包備 OKB。** 其他鏈先查官方 gas token。
- **v1.0 加快嘅係證明流水線，唔等於已用 FCR，亦唔等於大額可以取消最終性。** Katana 快速路徑同「最終性可選」都要顯式政策；未整合 FCR 之前，唔好把以太坊 `safe` 當 Agglayer 到帳訊號。
