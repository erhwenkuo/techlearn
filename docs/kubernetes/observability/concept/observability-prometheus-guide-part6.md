# Kubernetes 中的 Prometheus 指南

本文討論了在 Kubernetes 中安裝 Prometheus 的不同選項，然後使用 kube-prometheus-stack Helm chart詳細解釋了 Prometheus operator 的安裝。最後，我將介紹如何升級您的 Prometheus 堆棧以及您自己的警報規則和 Grafana 儀表板。

## 介紹

Prometheus 堆棧是一組流行的工具，用於實現系統的可觀察性。正如我在本 Observability 文章系列第一部分概述的學習指南中所解釋的那樣，Prometheus 堆棧可以在單個服務器上使用（例如與 Docker 一起使用），但旨在監控大型分佈式系統。 Prometheus 堆棧在 Kubernetes 中使用時會大放異彩，原因如下：

- Prometheus 帶有 Kubernetes 的動態服務發現 (Service Discovery)：在 Prometheus 配置文件的 `scrape_config` 部分中，您可以添加 `kubernetes_sd_config` 類型的目標，它指示 - Prometheus 聯繫 Kubernetes API 服務器以發現可用的節點、pod、服務等。 ，並從它們中提取指標。
- Kubernetes 的許多組件（例如 API 服務器）已經以 Prometheus 指標格式導出指標
- 在社群已經形成了一個大生態系統（kube-prometheus），同時也完善了 Kubernetes 和 Prometheus（以及相關組件，如 Alertmanager、Grafana 等）的集成。

!!! info "所需知識"
    要充分利用本文，您需要熟悉 Prometheus 和 Kubernetes：

    關於 Prometheus：首先需要對 Prometheus 要有很好的了解，包括它的架構、概念、如何配置、PromQL語言。

    關於 Kubernetes：你應該熟悉基本概念，包括 Helm 圖表和 Kubernetes operator。

在本文中，將討論不同的安裝選項，詳細解釋 Prometheus operator 的安裝，並總結如何升級 Prometheus 堆棧、警報規則和 Grafana 儀表板。

## Kubernetes 的 Prometheus 安裝選項

要在 Kubernetes 中啟動並運行 Prometheus，您有三個選擇：

1. 使用 Prometheus Helm chart 來安裝 Prometheus（以及一些相關組件，例如 Alertmanager，但不安裝 Grafana）。使用 Prometheus 原生文件（例如 `prometheus.yml` 或 `alert-rules.yml`）完成配置。

2. 使用 Helm chart 安裝 Prometheus operator，不使用任何 CR：不建議使用這種安裝方法，因為現在不推薦使用這種安裝方式，請改用選項 3。 Prometheus operator（它本身並沒有被棄用）是一個 Kubernetes operator，可讓您使用 CRD 處理供應和操作各種組件（例如 Prometheus 和 Alertmanager）。換句話說：配置是使用 Kubernetes 原生對象完成的，而不是 Prometheus 原生文件。 Prometheus operator 將它們即時轉換為 Alertmanager/Prometheus-native 配置格式。

3. 使用 kube-prometheus 包安裝 Prometheus Operator 以及 Grafana 和許多默認警報規則和儀表板。 [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) 是一個 GitHub 項目，其中包含一個巨大的最佳實踐庫和有關如何安裝 Prometheus 操作符的說明，包括許多合理的默認配置。

