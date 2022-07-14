# Agent Injector 對比 Vault CSI Provider

本文檔探討了將 HashiCorp Vault 與 Kubernetes 集成的兩種不同方法。提供的信息適用於了解機密管理概念並熟悉 HashiCorp Vault 和 Kubernetes 的 DevOps 從業者。本文檔還提供實用指南，幫助您了解和選擇最適合您的用例的方法。

本文檔中包含的信息詳細說明了 `Agent Injector`（在本文檔中也稱為 `Vault Sidecar` 或 `Sidecar`）與用於集成 Vault 和 Kubernetes 的 `Vault Container Storage Interface (CSI)` 提供程序之間的對比。

## Vault Sidecar Agent Injector

[Vault Sidecar Agent Injector](https://www.vaultproject.io/docs/platform/k8s/injector) 利用 [Sidecar 模式](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar)來更改 pod 規範以包含一個 Vault Agent 容器，該容器將 Vault 機密呈現到共享內存卷。通過將秘密呈現給共享卷，Pod 中的容器可以使用 Vault 秘密而無需 Vault 感知。注入器是一個 Kubernetes mutating webhook 控制器。如果請求中存在註釋，控制器會攔截 pod 事件並將變更應用於 pod。此功能由 [Vault-k8s](https://github.com/hashicorp/vault-k8s) 項目提供，可以使用 Vault Helm 圖表自動安裝和配置。

