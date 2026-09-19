# 可升級 Proxy：UUPS vs Transparent（CEX 視角）

## 人話（從你熟的發布）

CEX 自己寫充值／歸集合約，或者審「要唔要上某隻 proxy token」，都會撞到 **Proxy**：用戶以為 call 嘅係邏輯合約，其實先入 proxy，再 `delegatecall` 去 implementation。升級 = 換 implementation 指針，**storage 留喺 proxy**。  
兩大派：**Transparent**（admin 走升級、用戶走邏輯，靠「邊個 call」分流）同 **UUPS**（升級函數喺 implementation，靠 `upgradeTo` + 權限）。

## 面試短答

- **Proxy + delegatecall**：邏輯用 implementation 嘅 code，讀寫嘅係 proxy 嘅 storage → 升級唔搬資料，但 **storage layout 唔可以亂插槽**。
- **Transparent**：ProxyAdmin 專責升級；普通用戶 call 唔會撞到 admin 函數。多一個 admin 合約，gas／部署稍重。
- **UUPS**：`upgradeTo` 寫喺 impl；要有 `_authorizeUpgrade`；更省，但若新 impl 唔小心刪咗升級邏輯／權限錯，可能 **卡死無法再升級**（或被盜升級）。
- CEX 上幣／自研：checklist 必查 admin key 係邊、可唔可以 pause、有冇 timelock／multisig；storage 衝突同 selector clashing 係高頻追問。

## 常見追問（連答案）

**Q：點解「升級」好危險？**  
A：惡意／有 bug 嘅新 impl 可以改所有 storage（餘額映射、owner）。等同後門。CEX 對外部 proxy token：要當「發行方有超權」；對自研：admin 應放 multisig + timelock，唔好 EOA 單簽。

**Q：Storage layout 點保？**  
A：新版本只准 **append** 變數；唔好刪／改舊變數類型同順序。常用 OZ upgradeable 組件 + storage gap（`uint256[50] private __gap`）預留槽。面試可提「同繼承順序有關」。

**Q：同 CREATE2 factory 充值地址點共存？**  
A：Factory 可以 CREATE2 出 **Minimal Proxy（EIP-1167 clone）** 指向同一 impl，省 gas；若要可升級，clone 指向嘅係可升級 proxy 定固定 impl 要產品定。換 impl 影響所有 clone 行為——回歸同白名單要重新測。

## 小練習

**題：** 審計話某充值 token 係 UUPS，owner 係單個 EOA。CEX 風控問「上唔上」。你會列邊三條硬條件先考慮？

**參考答案：**  
(1) Owner 必須遷到 multisig（最好 + timelock），並核對 on-chain 已生效；(2) 有 pause／blacklist 等超權時，要有監控同「暫停充提」playbook，唔當普通 ERC-20；(3) 驗證 storage layout／歷史升級交易、當前 impl 代碼 hash 同文檔一致，並設 admin 變更／upgrade 事件告警。任一做唔到 → 拒上或只開「充值後立刻歸集、限制敞口」。
