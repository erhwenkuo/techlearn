# 概述

原文: [What Is Argo CD?](https://argo-cd.readthedocs.io/en/stable/)

## 什麼是 Argo CD?

Argo CD 是用於 Kubernetes 的聲明式 GitOps 持續交付工具。

![](./assets/argocd-ui.gif)

## 為什麼選擇 Argo CD?

應用程序定義、配置和環境應該是聲明式的和版本控制的。應用程序部署和生命週期管理應該是自動化的、可審計的並且易於理解。

## 運行的原理

Argo CD 遵循 GitOps 模式，使用 Git 存儲庫作為定義所需應用程序狀態的真實來源(source of true)。可以通過多種方式指定 Kubernetes 物件清單：

- [kustomize](https://kustomize.io/) applications
- [helm](https://helm.sh/) charts
- [jsonnet](https://jsonnet.org/) 文件
- YAML/json 清單的普通目錄
- 配置為配置管理插件的任何自定義配置管理工具

Argo CD 在指定的目標環境中自動部署所需的應用程序狀態。應用程序部署可以跟踪對分支、標籤的更新，或者在 Git 提交時固定到特定版本的清單。有關可用的不同跟踪策略的更多詳細信息，請參閱跟踪策略。

## 架構

![](./assets/argocd_architecture.png)

Argo CD 被實現為一個 kubernetes 控制器，它持續監控正在運行的應用程序並將當前的實時狀態與所需的目標狀態（如 Git 存儲庫中指定的）進行比較。實時狀態偏離目標狀態的已部署應用程序被視為 `OutOfSync`。 Argo CD 報告並可視化差異，同時提供自動或手動將實時狀態同步回所需目標狀態的工具。對 Git 存儲庫中所需目標狀態所做的任何修改都可以自動應用並反映在指定的目標環境中。

### 組件

#### API Server

API Server 是一個 gRPC/REST 服務器，它公開 Web UI、CLI 和 CI/CD 系統使用的 API。它具有以下職責：

- 應用程序管理和狀態報告
- 調用應用程序操作（例如同步、回滾、用戶定義的操作）
- 存儲庫和集群憑證管理（存儲為 K8s 秘密）
- 對外部身份提供者的身份驗證和授權委託
- RBAC 執行
- Git webhook 事件的偵聽器/轉發器

#### Repository Server

Repository Server 是一項內部服務，它維護一個包含應用程序清單的 Git 存儲庫的本地緩存。當提供以下輸入時，它負責生成和返回 Kubernetes 清單：

- repository URL
- revision (commit, tag, branch)
- application path
- template specific settings: parameters, helm values.yaml

#### Application Controller

Application Controller 是一個 Kubernetes 控制器，{==它持續監控正在運行的應用程序並將當前的實時狀態與所需的目標狀態（如 repo 中指定）進行比較==}。它檢測 **OutOfSync** 應用程序狀態並可選擇採取糾正措施。它負責為生命週期事件（PreSync、Sync、PostSync）調用任何用戶定義的掛鉤。

## 功能

- 將應用程序自動部署到指定的目標環境
- 支持多種配置管理/模板工具（Kustomize、Helm、Jsonnet、plain-YAML）
- 能夠管理和部署到多個集群
- SSO 集成（OIDC、OAuth2、LDAP、SAML 2.0、GitHub、GitLab、Microsoft、LinkedIn）
- 用於授權的多租戶和 RBAC 策略
- 回滾/隨地滾動到 Git 存儲庫中提交的任何應用程序配置
- 應用資源健康狀況分析
- 自動配置漂移檢測和可視化
- 自動或手動將應用程序同步到所需狀態
- 提供應用程序活動實時視圖的 Web UI
- 用於自動化和 CI 集成的 CLI
- Webhook 集成（GitHub、BitBucket、GitLab）
- 用於自動化的訪問令牌
- PreSync、Sync、PostSync 掛鉤以支持複雜的應用程序佈署（例如藍/綠和金絲雀升級）
- 應用程序事件和 API 調用的審計跟踪
- Prometheus 指標
- 用於在 Git 中覆蓋 helm 參數的參數覆蓋

## 核心概念

假設您熟悉核心 Git、Docker、Kubernetes、持續交付和 GitOps 概念。以下是一些特定於 Argo CD 的概念。

- **Application** 一組 Kubernetes 定義的資源清單。這是自定義資源定義 (CRD)。
- **Application source type** 用於構建應用程序工具。
- **Target state** 應用程序的期望狀態，由 Git 存儲庫中的文件表示。
- **Live state** 該應用程序的實時狀態。部署了哪些 pod 等。
- **Sync status** 實時狀態是否與目標狀態匹配。部署的應用程序是否與 Git 所說的一樣?
- **Sync** 使應用程序移動到其目標狀態的過程。例如。通過將更改應用於 Kubernetes 集群。
- **Sync operation status** 同步是否成功。
- **Refresh** 將 Git 中的最新代碼與實時狀態進行比較。弄清楚有什麼不同。
- **Health** 應用程序的健康狀況，是否正常運行？它可以滿足請求嗎？
- **Tool** 從文件目錄創建清單的工具。例如。定制化。請參閱應用程序源類型。
- **Configuration management tool** 請參見工具。
- **Configuration management plugin** 自定義工具。