# 歡迎來到 ArgoCD 教程

原文: https://redhat-scholars.github.io/argocd-tutorial/argocd-tutorial/index.html

## ArgoCD

[Argo CD](https://argoproj.github.io/argo-cd/) 是用於 Kubernetes 的聲明式 GitOps 持續交付工具。

它遵循 GitOps 模式，使用 Git 存儲庫作為定義所需應用程序狀態的事實來源。

它在指定的目標環境中自動部署所需的應用程序狀態。應用程序部署可以跟踪分支、標籤的更新，或在 Git 提交時固定到特定版本的清單。

![](./assets/argocd-logo.png)

## GitOps

[GitOps](https://www.openshift.com/learn/topics/gitops/) 是一組利用 Git 工作流來管理基礎設施和應用程序配置的實踐。通過使用 Git 存儲庫作為事實來源，它允許 DevOps 團隊將集群配置的整個狀態存儲在 Git 中，以便更改跟踪是可見和可審計的。

GitOps 通過將您的基礎設施和應用程序定義定義為“代碼”來簡化跨多個集群的基礎設施和應用程序配置更改的傳播。

- 確保集群具有相似的配置、監控或存儲狀態。
- 從已知狀態恢復或重新創建集群。
- 創建具有已知狀態的集群。
- 將配置更改應用或還原到多個集群。
- 將模板化配置與不同的環境相關聯。