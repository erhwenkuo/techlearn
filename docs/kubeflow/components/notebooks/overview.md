# 概述

原文: [Overview](https://www.kubeflow.org/docs/components/notebooks/overview/)

Kubeflow 筆記本 (Notebook) 概述

## 什麼是 Kubeflow 筆記本？

Kubeflow Notebooks 提供了一種在 Kubernetes 集群中運行基於 Web 的開發環境的方法，方法是在 Pod 中運行它們。

一些主要功能包括：

- 對 JupyterLab、RStudio 和 Visual Studio Code（代碼服務器）的原生支持。
- 用戶可以直接在集群中創建筆記本容器，而不是在他們的本地工作站上。
- 管理員可以為他們的組織提供標準的筆記本鏡像，並預先安裝所需的包或套件。
- 訪問控制由 Kubeflow 的 RBAC 管理，從而可以更輕鬆地在整個組織內共享筆記本。

