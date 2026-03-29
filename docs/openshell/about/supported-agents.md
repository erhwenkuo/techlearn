# Supported Agents

下表總結了在 OpenShell 沙箱中運行的 agent 程式。所有代理沙箱鏡像都維護在 OpenShell 社群倉庫中。當作為 `openshell sandbox create` 指令的尾部參數傳遞時，基礎鏡像中的代理會自動配置。

|Agent|來源|預設策略|備註|
|-----|---|-------|---|
|[Claude Code](https://docs.anthropic.com/en/docs/claude-code)|[base](https://github.com/NVIDIA/OpenShell-Community/tree/main/sandboxes/base)|Full coverage|開箱即用。需要 `ANTHROPIC_API_KEY`。|
|[OpenCode](https://opencode.ai/)|[base](https://github.com/NVIDIA/OpenShell-Community/tree/main/sandboxes/base)|Partial coverage|已預先安裝。若要使用全部功能，請將 `opencode.ai` 端點和 OpenCode 二進位檔案路徑新增至策略中。|
|[Codex](https://developers.openai.com/codex)|[base](https://github.com/NVIDIA/OpenShell-Community/tree/main/sandboxes/base)|No coverage|已預先安裝。需要包含 OpenAI 端點和 Codex 二進位檔案路徑的自訂策略。需要 `OPENAI_API_KE`Y。|
|[GitHub Copilot CLI](https://docs.github.com/en/copilot/github-copilot-in-the-cli)|[base](https://github.com/NVIDIA/OpenShell-Community/tree/main/sandboxes/base)|Full coverage|已預先安裝。開箱即用。需要 `GITHUB_TOKEN` 或 `COPILOT_GITHUB_TOKEN`。|
|[OpenClaw](https://openclaw.ai/)|[openclaw](https://github.com/NVIDIA/OpenShell-Community/tree/main/sandboxes/openclaw)|Bundled|代理編排層。使用 `openshell sandbox create --from openclaw` 啟動。|
|[Ollama](https://ollama.com/)|[ollama](https://github.com/NVIDIA/OpenShell-Community/tree/main/sandboxes/ollama)|Bundled|運行雲端和本地模型。包含 Claude Code, Codex 和 OpenCode。使用 `openshell sandbox create --from ollama` 啟動。|

[社區沙盒目錄](https://docs.nvidia.com/openshell/latest/sandboxes/community-sandboxes.html)中提供了更多社區代理沙盒。

如需查看完整的支援矩陣，請參閱[支援矩陣](https://docs.nvidia.com/openshell/latest/reference/support-matrix.html)頁面。