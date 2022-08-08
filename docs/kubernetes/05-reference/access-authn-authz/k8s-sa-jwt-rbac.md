# ServiceAccounts、JWT-tokens、身份驗證和 RBAC 授權

原文: https://rtfm.co.ua/en/kubernetes-serviceaccounts-jwt-tokens-authentication-and-rbac-authorization/

對於認證和授權，Kubernetes 有 User Accounts 和 Service Accounts 等概念。

User Accounts 用於從外部訪問集群的常用用戶配置文件，而 Service Accounts 用於從集群內部授予訪問權限。

ServiceAccounts 旨在為 **Kubernetes Pod** 提供一個身份，以供其容器在向 Kubernetes API 服務器執行 API 請求時對其進行身份驗證和授權。

## 預設的 ServiceAccount

每個 Kubernetes 命名空間都有自己的默認 ServiceAccount (SA)，它是在創建命名空間時創建的。

讓我們檢查一下 `default` 命名空間：

```bash
@ kubectl --namespace default get serviceaccount

NAME      SECRETS   AGE
default   1         44h
```

對於每個 ServiceAccount，都會生成一個 `token` 並將其存儲為 Kubernetes Secret。

讓我們檢查一下 `default` 的這個 ServiceAccount：

```bash hl_lines="12"
$ kubectl --namespace default get serviceaccount default -o yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-07-14T04:56:34Z"
  name: default
  namespace: default
  resourceVersion: "476"
  uid: cb81fb2a-a148-4a2a-b79f-89c101ce7c12
secrets:
- name: default-token-tvflf
```

在本範例中 SA 的令牌儲放在 `default-token-tvflf` 的 Secret 中：

!!! warning
    在 Kubernetes 1.24+ 之後的版本，K8s 不會再為 ServiceAccounts 自動生成 Secret。見 [BIG change in K8s 1.24 about ServiceAccounts and their Secrets](https://itnext.io/big-change-in-k8s-1-24-about-serviceaccounts-and-their-secrets-4b909a4af4e0)

### 預設的 token

現在，檢查 Secret 的內容：

```bash
$ kubectl get secret default-token-tvflf -o yaml

apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJkekNDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdGMyVnkKZG1WeUxXTmhRREUyTlRjM056UTFOell3SGhjTk1qSXdOekUwTURRMU5qRTJXaGNOTXpJd056RXhNRFExTmpFMgpXakFqTVNFd0h3WURWUVFEREJock0zTXRjMlZ5ZG1WeUxXTmhRREUyTlRjM056UTFOell3V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFSUkFMTldkL2ZwVGxwWGY1c1BndjVDaTZGUm5hcFBwYWZSb2V2ZTJ3dGYKdjFQYVRueGNERUVUM0FKbDVUdDd4RG9DRlRtMVRyWXRTN2MyTmFLOFd3bk5vMEl3UURBT0JnTlZIUThCQWY4RQpCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBZEJnTlZIUTRFRmdRVWh4OUhPeGJWQ0dRU2NZMC9ZaGxjCnlyQ25YcmN3Q2dZSUtvWkl6ajBFQXdJRFNBQXdSUUloQU5DQVRNOElScHNHaWs4R1FuWThGL1BEMFcrQXB6NW8KYzNqRnNXblZSc1B3QWlCdi9UWFQwSGtoZVpCZHJING0rZlUzeDNpU0UvVThqcGFGVEJ1ZUJqWUU0dz09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  namespace: ZGVmYXVsdA==
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklubDNOVlp3YWtSak1FWllVMjlHUVVOMlpFUmZVbWxuY0VSbFNsUlFjbEE1T1ZWcGJHeFlSMDkwYjI4aWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbVJsWm1GMWJIUXRkRzlyWlc0dGRIWm1iR1lpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pWkdWbVlYVnNkQ0lzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVnlkbWxqWlMxaFkyTnZkVzUwTG5WcFpDSTZJbU5pT0RGbVlqSmhMV0V4TkRndE5HRXlZUzFpTnpsbUxUZzVZekV3TVdObE4yTXhNaUlzSW5OMVlpSTZJbk41YzNSbGJUcHpaWEoyYVdObFlXTmpiM1Z1ZERwa1pXWmhkV3gwT21SbFptRjFiSFFpZlEuZ2dieDM4Z0FHUVI2VnNVSTJkYkdIWHF5VzItUGI2VjYzSUVCbVF0a2gyZy1pX3FOY3g0MHlGOVVZMjNjbHRtOXJlY0ZwTXVERTFlUlQ5THNhcFZuRHRPcGdPeW0yWjhqSzlGNXZLUjZrX0YyNVJvanRTdVFiOG9fb2t0RDJlTUlwemN2OTZuX1hDejdMcFdWUnRQMm0wUWJWTE03SGRoU2hOZHRfcTF3V3hIai1QYkJIOW5teXhhemdGeGE4bkhLczd2Y1JnTFBqOVRKRE03Qkg3U0VaOXlZOG41S1Zwbmt3djNSUEVsdlNaR0taci0xV0NRcTQ0NGVicVZWcVFpTE4wY2x1NTdhNk95N2tOTnRqR2x6SVhPb0pNWFpaQ0lZZERBbUFuSkNWa3hLRV9oR0xMME9nb2ZHMXdZUXVMWjltcVRBalc5RENQeDcxTl9VNmFLTEVn
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
    kubernetes.io/service-account.uid: cb81fb2a-a148-4a2a-b79f-89c101ce7c12
  creationTimestamp: "2022-07-14T04:56:34Z"
  name: default-token-tvflf
  namespace: default
  resourceVersion: "474"
  uid: 5cf55c44-2c6a-46a9-90ba-137cea5f27b9
type: kubernetes.io/service-account-token
```

- `type` - 標示了這個 Secret 的類型是 `kubernetes.io/service-account-token`。
- `data` - 保存了 `ca.ert` 與 `token` 的資訊 (base64 encoded)

`ca.crt` 由 K8S 集群的主密鑰簽名，因此集群扮演著證書頒發機構的角色，並允許 pod 或應用程序驗證 API-server。

現在，讓我們去研究 `token` 的部分。

### JWT token 內容

為了更容易從 termainal 工作 - 將 `data.token` 值保存到變量中：

```bash
$ token="ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklubDNOVlp3YWtSak1FWllVMjlHUVVOMlpFUmZVbWxuY0VSbFNsUlFjbEE1T1ZWcGJHeFlSMDkwYjI4aWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbVJsWm1GMWJIUXRkRzlyWlc0dGRIWm1iR1lpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pWkdWbVlYVnNkQ0lzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVnlkbWxqWlMxaFkyTnZkVzUwTG5WcFpDSTZJbU5pT0RGbVlqSmhMV0V4TkRndE5HRXlZUzFpTnpsbUxUZzVZekV3TVdObE4yTXhNaUlzSW5OMVlpSTZJbk41YzNSbGJUcHpaWEoyYVdObFlXTmpiM1Z1ZERwa1pXWmhkV3gwT21SbFptRjFiSFFpZlEuZ2dieDM4Z0FHUVI2VnNVSTJkYkdIWHF5VzItUGI2VjYzSUVCbVF0a2gyZy1pX3FOY3g0MHlGOVVZMjNjbHRtOXJlY0ZwTXVERTFlUlQ5THNhcFZuRHRPcGdPeW0yWjhqSzlGNXZLUjZrX0YyNVJvanRTdVFiOG9fb2t0RDJlTUlwemN2OTZuX1hDejdMcFdWUnRQMm0wUWJWTE03SGRoU2hOZHRfcTF3V3hIai1QYkJIOW5teXhhemdGeGE4bkhLczd2Y1JnTFBqOVRKRE03Qkg3U0VaOXlZOG41S1Zwbmt3djNSUEVsdlNaR0taci0xV0NRcTQ0NGVicVZWcVFpTE4wY2x1NTdhNk95N2tOTnRqR2x6SVhPb0pNWFpaQ0lZZERBbUFuSkNWa3hLRV9oR0xMME9nb2ZHMXdZUXVMWjltcVRBalc5RENQeDcxTl9VNmFLTEVn"
```

使用 `base64` 獲取其內容：

```bash
$ echo $token | base64 -d

eyJhbGciOiJSUzI1NiIsImtpZCI6Inl3NVZwakRjMEZYU29GQUN2ZERfUmlncERlSlRQclA5OVVpbGxYR090b28ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tdHZmbGYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImNiODFmYjJhLWExNDgtNGEyYS1iNzlmLTg5YzEwMWNlN2MxMiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.ggbx38gAGQR6VsUI2dbGHXqyW2-Pb6V63IEBmQtkh2g-i_qNcx40yF9UY23cltm9recFpMuDE1eRT9LsapVnDtOpgOym2Z8jK9F5vKR6k_F25RojtSuQb8o_oktD2eMIpzcv96n_XCz7LpWVRtP2m0QbVLM7HdhShNdt_q1wWxHj-PbBH9nmyxazgFxa8nHKs7vcRgLPj9TJDM7BH7SEZ9yY8n5KVpnkwv3RPElvSZGKZr-1WCQq444ebqVVqQiLN0clu57a6Oy7kNNtjGlzIXOoJMXZZCIYdDAmAnJCVkxKE_hGLL0OgofG1wYQuLZ9mqTAjW9DCPx71N_U6aKLEg
```

JWT 的 token 包含了三部份的資訊:

- 標頭 `header` - 描述令牌的簽名方式
- 有效負載 `payload` - 令牌的實際數據，例如到期日期、頒發者等，請參閱 [RFC-7519](https://tools.ietf.org/html/rfc7519#section-4.1)
- 簽名 `signature` - 用於驗證令牌沒有被修改，並可用於驗證發件人

要檢查令牌的內容，我們可以使用 jwtutility 或在 [jwt.io](https://jwt.io/) 網站。

![](./assets/k8s-sa-jwt.png)

有效負載 `payload` 部分有以下資訊：

```json
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "default",
  "kubernetes.io/serviceaccount/secret.name": "default-token-tvflf",
  "kubernetes.io/serviceaccount/service-account.name": "default",
  "kubernetes.io/serviceaccount/service-account.uid": "cb81fb2a-a148-4a2a-b79f-89c101ce7c12",
  "sub": "system:serviceaccount:default:default"
}
```

- `iss` - 令牌的簽發者 issuer
- `kubernetes.io/serviceaccount/namespace` - 標示了令牌歸屬於那個命名空間
- `kubernetes.io/serviceaccount/secret.name` - 標示令牌代表者那一個服務帳戶
- `kubernetes.io/serviceaccount/service-account.uid` - 服務帳戶的內部 id
- `sub` - 標示了一個 unique 的完整服務帳戶的名稱


## ServiceAccount 和 RBAC

對於沒有指定 ServiceAccount 的每個 Pod，Kubernetes 都會自動指派使用 `default` 這個 ServiceAccount 並掛載了它的 JWT 令牌。

回到我們的 Kubernetes 集群並運行一個 Pod：

```bash
$ kubectl run -i --tty --rm ca-test-pod --image=radial/busyboxplus:curl

[ root@ca-test-pod:/ ]$ 
```

在 pod 內部檢查 `/var/run/secrets/kubernetes.io/serviceaccount` 目錄內容：

```bash hl_lines="3-5"
[ root@ca-test-pod:/ ]$ ls -1 /var/run/secrets/kubernetes.io/serviceaccount

ca.crt
namespace
token
```

在 pod 內部檢查環境變數：

```bash hl_lines="3-5"
[ root@ca-test-pod:/ ]$ env | grep KUBERNETES

KUBERNETES_PORT=tcp://10.43.0.1:443
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.43.0.1
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.43.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.43.0.1
```

嘗試在沒有身份驗證的情況下向 API-server 執行請求 - 使用特殊的 Service `kubernetes`，將 `-k` 或 `--insecure` 添加到 curl 以跳過服務器的證書驗證

```bash
[ root@ca-test-pod:/ ]$ curl -k https://kubernetes

{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

我們得到 `401` Unauthorized。

現在讓我們在呼叫設定上添加兩個變量 `ca.crt` 和 `token`：

```bash
[ root@ca-test-pod:/ ]$ CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
[ root@ca-test-pod:/ ]$ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

再次運行 curl, 讓我們嘗試獲取命名空間中的 pod 列表，這次不使用 --insecure 並使用 Authorization 標頭進行授權：

```bash
[ root@ca-test-pod:/ ]$ curl --cacert $CERT -H "Authorization: Bearer $TOKEN" "https://kubernetes/api/v1/namespaces/default/pods/"

{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:default:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

這次我們可以看到我們的用戶 `system:serviceaccount:default:default` 沒有權限來呼叫本此的API。

## ServiceAccount 與 RoleBindig

為了給我們的 SericeAccount 權限，我們需要像普通用戶一樣創建 `RoleBinding` 或 `ClusterRoleBinding`。

創建到默認 ClusterRole 視圖的 RoleBinding 映射，請參閱[User-facing roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles)：

```bash
$ kubectl create rolebinding ca-test-view --clusterrole=view --serviceaccount=default:default

rolebinding.rbac.authorization.k8s.io/ca-test-view created
```

並再次運行 curl：

```bash
[ root@ca-test-pod:/ ]$ curl --cacert $CERT -H "Authorization: Bearer $TOKEN" "https://kubernetes/api/v1/namespaces/default/pods/"
```

結果:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:default:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}[ root@ca-test-pod:/ ]$ curl --cacert $CERT -H "Authorization: Bearer $TOKEN" "https://kubernetes/api/v1/namespaces/default/pods/"
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "14260"
  },
  "items": [
    {
      "metadata": {
        "name": "ca-test-pod",
        "namespace": "default",
        "uid": "db7b286c-a31d-4697-bed6-7e75cb7865e9",
        "resourceVersion": "13958",
        "creationTimestamp": "2022-07-16T01:50:29Z",
        "labels": {
          "run": "ca-test-pod"
        },
        "managedFields": [
          {
            "manager": "kubectl-run",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2022-07-16T01:50:29Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {"f:metadata":{"f:labels":{".":{},"f:run":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"ca-test-pod\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:stdin":{},"f:stdinOnce":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:tty":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}
          },
          {
            "manager": "k3s",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2022-07-16T01:50:38Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"10.42.0.11\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},
            "subresource": "status"
          }
        ]
      },
      "spec": {
        "volumes": [
          {
            "name": "kube-api-access-cngn5",
            "projected": {
              "sources": [
                {
                  "serviceAccountToken": {
                    "expirationSeconds": 3607,
                    "path": "token"
                  }
                },
                {
                  "configMap": {
                    "name": "kube-root-ca.crt",
                    "items": [
                      {
                        "key": "ca.crt",
                        "path": "ca.crt"
                      }
                    ]
                  }
                },
                {
                  "downwardAPI": {
                    "items": [
                      {
                        "path": "namespace",
                        "fieldRef": {
                          "apiVersion": "v1",
                          "fieldPath": "metadata.namespace"
                        }
                      }
                    ]
                  }
                }
              ],
              "defaultMode": 420
            }
          }
        ],
        "containers": [
          {
            "name": "ca-test-pod",
            "image": "radial/busyboxplus:curl",
            "resources": {
              
            },
            "volumeMounts": [
              {
                "name": "kube-api-access-cngn5",
                "readOnly": true,
                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
              }
            ],
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "imagePullPolicy": "IfNotPresent",
            "stdin": true,
            "stdinOnce": true,
            "tty": true
          }
        ],
        "restartPolicy": "Always",
        "terminationGracePeriodSeconds": 30,
        "dnsPolicy": "ClusterFirst",
        "serviceAccountName": "default",
        "serviceAccount": "default",
        "nodeName": "k3d-k3s-default-server-0",
        "securityContext": {
          
        },
        "schedulerName": "default-scheduler",
        "tolerations": [
          {
            "key": "node.kubernetes.io/not-ready",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          },
          {
            "key": "node.kubernetes.io/unreachable",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          }
        ],
        "priority": 0,
        "enableServiceLinks": true,
        "preemptionPolicy": "PreemptLowerPriority"
      },
      "status": {
        "phase": "Running",
        "conditions": [
          {
            "type": "Initialized",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2022-07-16T01:50:29Z"
          },
          {
            "type": "Ready",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2022-07-16T01:50:38Z"
          },
          {
            "type": "ContainersReady",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2022-07-16T01:50:38Z"
          },
          {
            "type": "PodScheduled",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2022-07-16T01:50:29Z"
          }
        ],
        "hostIP": "172.24.0.2",
        "podIP": "10.42.0.11",
        "podIPs": [
          {
            "ip": "10.42.0.11"
          }
        ],
        "startTime": "2022-07-16T01:50:29Z",
        "containerStatuses": [
          {
            "name": "ca-test-pod",
            "state": {
              "running": {
                "startedAt": "2022-07-16T01:50:37Z"
              }
            },
            "lastState": {
              
            },
            "ready": true,
            "restartCount": 0,
            "image": "docker.io/radial/busyboxplus:curl",
            "imageID": "sha256:4776f1f7d1f625c8c5173a969fdc9ae6b62655a2746aba989784bb2b7edbfe9b",
            "containerID": "containerd://c3e6ffab06622a2506ed6cf18ec8acd61fbb2a45721a2d65617b5bbce2a91159",
            "started": true
          }
        ],
        "qosClass": "BestEffort"
      }
    }
  ]
}
```
