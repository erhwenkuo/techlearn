# Default Policy Reference

預設政策是指在建立 OpenShell 沙箱時不使用 `--policy` 參數時所套用的政策。它已整合到 community base image ([`ghcr.io/nvidia/openshell-community/sandboxes/base`](https://github.com/nvidia/openshell-community)) 中，並在社區倉庫的 `dev-sandbox-policy.yaml` 檔案中定義。

## Agent Compatibility

下表顯示了常用代理程式的預設策略覆蓋範圍。

|Agent|範圍|所需採取的行動|
|-----|---|------------|
|**Claude Code**|Full|沒有。開箱即用。|
|**OpenCode**|Partial|新增 `opencode.ai` 端點和 OpenCode 二進位檔案路徑。|
|**Codex**|None|提供包含 OpenAI 端點和 Codex 二進位檔案路徑的完整自訂策略。|

!!! info
    如果您執行的非 Claude 代理程式沒有自訂策略，代理程式的 API 呼叫將被代理拒絕。您必須提供一個策略來聲明代理程式的端點和二進位檔案。

## Default Policy Blocks

預設策略區塊定義在社群基礎鏡像中。完整的 `dev-sandbox-policy.yaml` 原始碼請參閱 [openshell-community 倉庫](https://github.com/nvidia/openshell-community)。

