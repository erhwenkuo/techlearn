# 修剪

原文:[Prune](https://sungwook-choi.gitbook.io/argocd/sync/prune)

## 概念

**prune** 是刪除同步資源的選項。當您將 Kubernetes 資源與 argocd 同步並從 git 中刪除資源時，該資源將從 Kubernetes 中刪除。

如果關閉 prune (prune = false)，與 argocd 同步的 Kubernetes 資源將保留在 Kubernetes 中，即使資源在 git 中被刪除。反之，開啟 prune 後，git 中刪除 Kubernetes 資源時也會刪除。

## 啟動

默認情況下不啟用 prune 選項。創建應用程序時，您必須點選 `Prune` 複選框以將其啟動。

