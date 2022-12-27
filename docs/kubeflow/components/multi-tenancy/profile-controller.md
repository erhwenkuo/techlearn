# Kubeflow Profile (CRD)

原文:[Kubeflow Profile](https://github.com/kubeflow/kubeflow/blob/master/components/profile-controller/README.md)

Kubeflow Profile CRD 旨在解決多用戶 kubernetes 集群中的訪問管理。

配置文件訪問管理提供命名空間級別的隔離，基於：

- Kubernetes RBAC
- Istio 授權策略

## Profile CRD 管理的 Resources

每個 `profile (CRD)` 將管理一個命名空間（與配置文件 CRD 同名）並有一個 User (owner) 所有者。具體來說，每個 `profile (CRD)` 將管理以下資源：

- Namespace 保留給 profile owner.
- K8s RBAC RoleBinding `namespaceAdmin`: 設定 profile owner 成為 namespace 管理員，允許通過 k8s API (kubectl) 訪問上述命名空間。
- Istio namespace-scoped ServiceRole `ns-access-istio`: 允許通過 Istio 路由訪問目標命名空間中的所有服務。
- Istio namespace-scoped ServiceRoleBinding `owner-binding-istio`: 將 ServiceRole `ns-access-istio` 綁定到 profile owner 。因此 profile owner 可以通過 Istio（瀏覽器）訪問上述命名空間中的服務。
- 設置命名空間範圍內的服務帳戶 `editor` 和 `viewer` 以供上述命名空間中用戶創建的 pod 使用。
- Resource Quota
- Custom Plugins

## 管理訪問控制和資源

集群管理員可以管理集群用戶的訪問管理：

範例#01: 為用戶 `user1@abcd.com` 創建隔離的命名空間 `kubeflow-user1`

管理員可以通過 kubectl 創建 `profile` 物件:

```bash
kubectl create -f /path/to/profile/config
```

範例#02: 撤銷用戶 `user1@abcd.com` 對命名空間 `kubeflow-user1` 的訪問並刪除命名空間 `kubeflow-user1`

管理員可以刪除 `kubeflow-user1` 的 `profile` 物件:

```bash
kubectl delete profile kubeflow-user1
```

## 用戶自主註冊 kfam UI

有權訪問集群 API 服務器的用戶應該能夠在沒有管理員手動批准的情況下註冊和使用 kubeflow 集群。

## Profile CRD 的結構

```yaml
apiVersion: kubeflow.org/v1
kind: Profile
metadata:
  name: test-user-profile
spec:
  owner:
    kind: User
    name: test-user@kubeflow.org
```

```yaml
apiVersion: kubeflow.org/v1beta1
kind: Profile
metadata:
  name: profileName   # replace with the name of profile you want, this will be user's namespace name
spec:
  owner:
    kind: User
    name: userid@email.com   # replace with the email of the user

  resourceQuotaSpec:    # resource quota can be set optionally
    hard:               # hard is the set of desired hard limits for each named resource. 
      requests.cpu: "2"             # Across all pods, the sum of CPU requests cannot exceed this value.
      requests.memory: 2Gi          # Across all pods, the sum of memory requests cannot exceed this value.
      requests.nvidia.com/gpu: "1"  # Across all pods, the sum of nvidia GPU requests cannot exceed this value.
      persistentvolumeclaims: "1"   # The total number of PersistentVolumeClaims that can exist in the namespace.
      requests.storage: "5Gi"       # Across all persistent volume claims, the sum of storage requests cannot exceed this value.
```


```yaml
apiVersion: kubeflow.org/v1
kind: Profile
metadata:
  name: profile-aws-iam
spec:
  owner:
    kind: User
    name: test-user@kubeflow.org
  plugins:
  - kind: AwsIamForServiceAccount
    spec:
      awsIamRole: arn:aws:iam::account-id:role/s3-reader
```
