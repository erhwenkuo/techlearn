# Self-host standalone servers

本指南將向您展示如何在不使用 LangSmith 使用者介面或控制平面的情況下部署 **standalone Agent Servers**。這是運行一個或幾個 agents 作為獨立服務的最輕量級自託管方案。

!!! warning
    這種部署方式提供了靈活性，但需要您自行管理基礎架構和配置。

    對於 production workloads，可以考慮 [self-hosting the full LangSmith platform](https://docs.langchain.com/langsmith/self-hosted) 或使用 [deploying with the control plane](https://docs.langchain.com/langsmith/deploy-with-control-plane) 進行部署，它們提供標準化的部署模式和基於 UI 的管理。

!!! info
    這是直接部署 Agent Servers（不使用 LangSmith 平台）的設定頁面。

    查看 [self-hosted options](https://docs.langchain.com/langsmith/self-hosted) 以了解:

    - [Standalone Server](https://docs.langchain.com/langsmith/self-hosted#standalone-server): 本指南涵蓋的內容（不涉及使用者介面，僅涉及伺服器）。
    - [LangSmith](https://docs.langchain.com/langsmith/self-hosted#langsmith): 適用於具有使用者介面的完整 LangSmith 平台。
    - [LangSmith Deployment](https://docs.langchain.com/langsmith/self-hosted#langsmith-deployment): 用於基於使用者介面的部署管理。

    在繼續之前，請先查看 [standalone server overview](https://docs.langchain.com/langsmith/self-hosted#standalone-server)。

## Prerequisites