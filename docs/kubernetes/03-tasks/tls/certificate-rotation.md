# 為 kubelet 配置證書輪換

原文:[Configure Certificate Rotation for the Kubelet](https://kubernetes.io/docs/tasks/tls/certificate-rotation/)

本文展示如何在 kubelet 中啟用並配置證書輪換。

## 準備開始

- Kubernetes 1.8.0 或更高的版本

## 概述

Kubelet 使用證書進行 Kubernetes API 的認證。默認情況下，這些證書的簽發期限為一年，所以不需要太頻繁地進行更新。

Kubernetes 包含特性 kubelet 證書輪換， 在當前證書即將過期時， 將自動生成新的秘鑰，並從 Kubernetes API 申請新的證書。一旦新的證書可用，它將被用於與 Kubernetes API 間的連接認證。

## 啟用客戶端證書輪換

`kubelet` 進程接收 `--rotate-certificates` 參數，該參數決定 kubelet 在當前使用的 證書即將到期時，是否會自動申請新的證書。

`kube-controller-manager` 進程接收 `--cluster-signing-duration` 參數，用來控制簽發證書的有效期限。

## 理解證書輪換配置

當 kubelet 啟動時，如被配置（使用`--bootstrap-kubeconfig` 參數），kubelet 會使用其初始證書連接到 Kubernetes API ，並發送證書簽名的請求。可以通過以下方式查看證書簽發請求 (CSR) 的狀態：

```bash
kubectl get csr
```

最初，來自節點上 kubelet 的證書簽名請求 (csr) 處於 `Pending` 狀態。如果證書簽名請求滿足特定條件， 控制器管理器會自動批准，此時請求會處於 `Approved` 狀態。接下來，控制器管理器會簽署證書， 證書的有效期限由 `--cluster-signing-duration` 參數指定，簽署的證書會被附加到證書簽名請求中。

Kubelet 會從 Kubernetes API 取回簽署的證書，並將其寫入磁盤，存儲位置通過 `--cert-dir` 參數指定。然後 kubelet 會使用新的證書連接到 Kubernetes API。

當簽署的證書即將到期時，kubelet 會使用 Kubernetes API，自動發起新的證書簽名請求。該請求會發生在證書的有效時間剩下 30% 到 10% 之間的任意時間點。同樣地，控制器管理器會自動批准證書請求，並將簽署的證書附加到證書簽名請求中。 Kubelet 會從 Kubernetes API 取回簽署的證書，並將其寫入磁盤。然後它會更新與 Kubernetes API 的連接，使用新的證書重新連接到 Kubernetes API。

