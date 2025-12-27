# Deep Agents overview

建構能夠規劃、使用 sub-agent 並利用檔案系統處理複雜任務的 agents。

[deepagents](https://pypi.org/project/deepagents/) 是一個獨立的函式庫，用於建立能夠處理複雜多步驟任務的 agents。 deepagents 是基於 LangGraph 構建，並受到 Claude Code, Deep Research 和 Manus 等應用程式的啟發，具備規劃功能、用於上下文管理的檔案系統以及生成子代理程式的能力。

## When to use deep agents

當您需要具備以下功能的代理程式時，請使用 deep agents：

- 處理需要規劃和分解的複雜多步驟任務
- 透過檔案系統工具管理大量上下文訊息
- 將工作委派給專門的子代理程式以實現上下文隔離
- 跨對話和線程持久化內存

對於更簡單的用例，可以考慮使用 LangChain 的 `create_agent` 或建立自訂 LangGraph 工作流程。


## Core capabilities

### Planning and task decomposition

Deep agents 包含一個內建的 `write_todos` 工具，使智能體能夠將複雜的任務分解為離散的步驟，追蹤進度，並隨著新資訊的出現調整計劃。

### Context management

檔案系統工具（`ls`, `read_file`, `write_file`, `edit_file`）允許代理程式將大型上下文卸載到記憶體中，防止上下文視窗溢出，並能夠處理可變長度的工具結果。

### Subagent spawning

內建的 `task` 工具使智能體能夠產生專門的 subagents，從而實現上下文隔離。這既能保持主智能體上下文的清晰，又能使其深入處理特定的子任務。

### Long-term memory

使用 LangGraph 的 Store 為代理程式添加跨線程的持久記憶體。代理可以保存和檢索先前對話中的信息。

## Relationship to the LangChain ecosystem

Deep agents 建構於以下基礎：

- **LangGraph** - 提供底層圖執行與狀態管理
- **LangChain** - 工具和模型整合與深度代理無縫協作
- **LangSmith** - 可觀測性、評估和部署

Deep agents 應用程式可以透過 LangSmith Deployment 進行部署，並使用 LangSmith Observability 進行監控。