# Kubeflow 介詔

Kubeflow 項目致力於使機器學習 (ML) 工作流在 Kubernetes 上的部署變得簡單、可移植和可擴展。目標不是重新創建其他服務，而是提供一種直接的方法，將用於 ML 的同類最佳開源系統部署到不同的基礎設施中。在任何運行 Kubernetes 的地方，您都應該能夠運行 Kubeflow。

## 開始使用 Kubeflow

閱讀[架構概述](./architecture.md)以了解 Kubeflow 架構的介紹，並了解如何使用 Kubeflow 來管理您的 ML 工作流。

按照[安裝 Kubeflow](./kubeflow-install.md) 設置您的環境並安裝 Kubeflow。

觀看以下視頻，其中介紹了 Kubeflow。

![type:video](https://www.youtube.com/embed/cTZArDgbIWw)

## 什麼是 Kubeflow？

Kubeflow 是 Kubernetes 的機器學習工具包。

要使用 Kubeflow，基本工作流程是：

- 下載部署並運行 Kubeflow。
- 自定義生成的配置文件。
- 運行指定的腳本以將您的容器部署到您的特定環境。

您可以調整配置以選擇要用於 ML 工作流程每個階段的平台和服務：

1. 數據準備
2. 模型訓練，
3. 推論服務
4. 服務管理

您可以選擇在本地、地端資料中心或雲環境中部署 Kubernetes 與 Kubeflow。

## Kubeflow 使命

Kubeflow 的目標是通過讓 Kubernetes 做它擅長的事情，盡可能簡單地擴展機器學習 (ML) 模型並將它們部署到生產環境中：

- 在不同的基礎設施上進行簡單、可重複、便攜的部署（例如，在筆記本電腦上進行試驗，然後遷移到本地集群或雲端）
- 部署和管理鬆散耦合的微服務
- 按需來擴展資源


由於 ML 開發團隊使用多種不同工具，其中一個關鍵目標是根據用戶需求（在合理範圍內）自定義堆棧，讓系統處理“無聊的事情”。雖然我們從一組狹窄的技術開始，但我們正在與許多不同的項目合作以包括額外的工具。

最終，我們希望有一組簡單的清單，為 ML 開發團隊提供一個易於使用的 ML 堆棧，無論 Kubernetes 已經運行在哪裡，並且可以根據它部署到的集群進行配置調整與客製。
