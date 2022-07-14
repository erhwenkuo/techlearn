# Working with Kubernetes API

原文: https://iximiuz.com/en/series/working-with-kubernetes-api/

Kubernetes 已經成為我們許多人新一代基礎設施的通用語。在很大程度上，這是由於 Kubernetes 提供了可移植的 API 來管理服務器資源。就像 POSIX 定義了消耗單台計算機資源的標準方式一樣，Kubernetes API 以與提供者無關的方式啟用了數據中心資源的消耗。

這是關於從命令行和代碼使用 Kubernetes API 的系列文：

1. 本系列首先概述最基本的 API 概念。
2. 解釋了Resource、Kind​​和Object等關鍵的基礎知識。
3. 說明 API 結構和術語。
4. 有大量關於如何訪問和擴展 API 的實際示例。

這系列還可以幫助讀者了解當代 Kubernetes API 客戶端（Go 和 Rust），從基本的 REST 功能開始，到 Informers 和 Work Queues 等高級抽象，因此它對編寫各種 Kubernetes 自動化的人很有用包括自定義 custom operator 和 controller。

- [Kubernetes API Basics - Resources, Kinds, and Objects](https://iximiuz.com/en/posts/kubernetes-api-structure-and-terminology/)
- [How To Call Kubernetes API using Simple HTTP Client](https://iximiuz.com/en/posts/kubernetes-api-call-simple-http-client/)
- [How To Call Kubernetes API using Go - Types and Common Machinery](https://iximiuz.com/en/posts/kubernetes-api-go-types-and-common-machinery/)
- [How To Extend Kubernetes API - Kubernetes vs. Django](https://iximiuz.com/en/posts/kubernetes-api-how-to-extend/)
- [How To Develop Kubernetes CLIs Like a Pro](https://iximiuz.com/en/posts/kubernetes-api-go-cli/)

---

![](./assets/k8s-api-intro.png)

