# Troubleshooting

Kubeflow Notebooks 常見問題及解決方案。

## 問題: notebook not starting

### SOLUTION: check events of Notebook

運行以下命令，然後檢查事件部分以確保沒有錯誤：

```bash
kubectl describe notebooks "${MY_NOTEBOOK_NAME}" --namespace "${MY_PROFILE_NAMESPACE}"
```

### SOLUTION: check events of Pod

運行以下命令，然後檢查事件部分以確保沒有錯誤：

```bash
kubectl describe pod "${MY_NOTEBOOK_NAME}-0" --namespace "${MY_PROFILE_NAMESPACE}"
```

### SOLUTION: check YAML of Pod

運行以下命令並檢查 Pod YAML 是否符合預期：

```bash
kubectl get pod "${MY_NOTEBOOK_NAME}-0" --namespace "${MY_PROFILE_NAMESPACE}" -o yaml
```

### SOLUTION: check logs of Pod

運行以下命令從 Pod 獲取日誌：

```bash
kubectl logs "${MY_NOTEBOOK_NAME}-0" --namespace "${MY_PROFILE_NAMESPACE}"
```

## 問題: manually delete notebook

### SOLUTION: use kubectl to delete Notebook resource

運行以下命令以手動刪除筆記本資源：

```bash
kubectl delete notebook "${MY_NOTEBOOK_NAME}" --namespace "${MY_PROFILE_NAMESPACE}"
```
