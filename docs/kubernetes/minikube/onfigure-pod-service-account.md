# 配置 Pod 的 Service Account

原文: https://jimmysong.io/kubernetes-handbook/guide/configure-pod-service-account.html

`Service account` 為 Pod 中的進程提供身份信息。

本文是關於 **Service Account 的用戶指南**，管理指南另見 **Service Account 的集群管理指南** 。

注意：本文檔描述的關於 Service Account 的行為只有當您按照 Kubernetes 項目建議的方式搭建起集群的情況下才有效。您的集群管理員可能在您的集群中有自定義配置，這種情況下該文檔可能並不適用。

當您（真人用戶）訪問集群（例如使用kubectl命令）時，apiserver 會將您認證為一個特定的 User Account（目前通常是admin，除非您的系統管理員自定義了集群配置）。 Pod 容器中的進程也可以與 apiserver 聯繫。當它們在聯繫 apiserver 的時候，它們會被認證為一個特定的 Service Account（例如default）。

## 使用默認的 Service Account 訪問 API server

當您創建 pod 的時候，如果您沒有指定一個 service account，系統會自動得在與該pod 相同的 namespace 下為其指派一個default service account。如果您獲取剛創建的 pod 的原始 json 或 yaml 信息（例如使用`kubectl get pods/podename -o yaml`命令），您將看到`spec.serviceAccountName`字段已經被設置為 `automatically`。

## 使用 Service Account 作為用戶權限管理配置 kubeconfig

### 創建服務賬號

```bash
$ kubectl create serviceaccount sample-sc
```

這時候我們將得到一個在 `default` namespace 的 serviceaccount 賬號； 我們運行 `kubectl get serviceaccount sample-sc -o yaml` 將得到如下結果:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-06-25T04:15:41Z"
  name: sample-sc
  namespace: default
  resourceVersion: "10098"
  uid: cc45957d-3b67-4382-ac0b-c49ebfbcb3f6
secrets:
- name: sample-sc-token-wmdvw
```

因為我們在使用 serviceaccount 賬號配置 kubeconfig 的時候需要使用到 sample-sc 的 token， 該 token 保存在該 serviceaccount 保存的 secret 中；

我們運行 `kubectl get secret sample-sc-token-wmdvw -o yaml` 將會得到如下結果:

```yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCakNDQWU2Z0F3SUJBZ0lCQVRBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwdGFXNXAKYTNWaVpVTkJNQjRYRFRJeU1EWXhNakEyTVRReE5Gb1hEVE15TURZeE1EQTJNVFF4TkZvd0ZURVRNQkVHQTFVRQpBeE1LYldsdWFXdDFZbVZEUVRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTUx1ClJibkdZeDhWMWVjaVdwSkloaVBiREtJTTVlbTBteWwxQk1EaUsvS09Lb3BIQm02QjBCT1FDZ2h4bVNNQlYwZWIKSFFDUGxWQWdseE5zTWtLMUJNZzZDMTB3bkZXOUtGTWJoOTRFcHdRWTZSWTRSemlxcHdnMUhHTEQ1UzJKcmd2agpPNTIrb3JnNzVvR3NkUkwrOTV1aEc2aEc5NTI0emtFcmxkNllHMUttV2JsVXptRFc4eXVsdUVQaHlvZGNqR0g0CldVdEhISE1nclQyTytnNml4YmN0aTVxRlAydzVSaGJvVDZtUjRvWUM4eTFtZld4MFppTjZOQ1NGMmZZZ3NEbDkKYUliUVovR1EzNnU5VW9WY3U1bWdxN21sdlI0SWFZNSs0dURWOHFWQWR1Y0QzcXJqK1JZUnllMmpqNXJEU0VRZwpYa3lORS9xZzkyaUc5WTlmbHo4Q0F3RUFBYU5oTUY4d0RnWURWUjBQQVFIL0JBUURBZ0trTUIwR0ExVWRKUVFXCk1CUUdDQ3NHQVFVRkJ3TUNCZ2dyQmdFRkJRY0RBVEFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQjBHQTFVZERnUVcKQkJSd1U0YU9BYTdqVDFlUHNERERNYkJlTWR0L2xqQU5CZ2txaGtpRzl3MEJBUXNGQUFPQ0FRRUFWanplV1FuTwplNytTc1AxRXl1ZURTS0w0cHowQnNtSlFZc3ZBZkROZEphTDBScStJc0FMYmhPWk15Nk4wUnUzdmxrZ1YrY3ZYCkIwSU9zRCsvdlN2bUFtYWNwclJjM3ZrS1cyak1lZy9xR3JWTy8vK1Y5N25Fb1IvbGRtTDB4NEl0bmFaNW1aWHkKOTNkWUVZY0tkb2pqTXY1S0dKbjlvcmhRS2Z0SzlYUk1zZ3kxUW5XVGJMbUEyeENMUGtuQzhoV0U1QS85bXlFRgo3UTVtaFFSWS85akQ2ZVJFeERBbytuKzRlVHlPd3loa3pXenEyMWVpNmJNbUt0SVlUdlc1SFNOSXo3NzNoV3grCjNjaUZlSzZTcHg0c0RpckdYenFjaEk4dHpTQ3NNS1o1ODAwSWp1T1RNb0VCM01WN0Rhalk0eE1MMW1nZlBta1EKRmtxZWQwcGhqNVRsUEE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  namespace: ZGVmYXVsdA==
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltUk9hbkJKYmpGd2JVbElRbHByU1VKb2VrdExTV2xoZFRBMk9IWkhXbTFWVlVsNWF6SnRRMGgyWjBFaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbk5oYlhCc1pTMXpZeTEwYjJ0bGJpMTNiV1IyZHlJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZ5ZG1salpTMWhZMk52ZFc1MExtNWhiV1VpT2lKellXMXdiR1V0YzJNaUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNTFhV1FpT2lKall6UTFPVFUzWkMwellqWTNMVFF6T0RJdFlXTXdZaTFqTkRsbFltWmlZMkl6WmpZaUxDSnpkV0lpT2lKemVYTjBaVzA2YzJWeWRtbGpaV0ZqWTI5MWJuUTZaR1ZtWVhWc2REcHpZVzF3YkdVdGMyTWlmUS5DNVdxM2w0NjFWRTdPbkdoUW9TaU5XQm5yQ0hwVFV1VnZueHlBZ2JXTDNmYU9tSmwxTDhXWGtBSUlGbGtYR1UyLUx1VlpfYjJjX1BMUUNFbm9NZ016TFBSWGFncXFzZ1FtLXZSdnFNVlgtUUlwZjc3eEZJeV9mN3k0MWVYaXhuWVZ1UWZZZk42TUpHVFJoYnhMZ25COUstY3A1dnNtUkRWUUtqZ3BoU3ZybTFmRllTNW1NUzY0cjJSYnFQMWhxc1h3c2pvbW9FS2daSlRGamt5UWdVaFhxb1ZqT250d3NGaU53NVBaYUEwZGdocEY3RWNvMF9VSVZOQTVCa0dkYkI0ak9mbDU2TUNVUk1oMU1iVVdhaHFkMnJ4Q1E2bXk0c0ZOcE8tOEstQmtkMEpOWF9yNDBoSjF4MHFPUlBmSXpvVk1Oc0Exdl90cTQ1c3pwN3BjRGxjbWc=
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: sample-sc
    kubernetes.io/service-account.uid: cc45957d-3b67-4382-ac0b-c49ebfbcb3f6
  creationTimestamp: "2022-06-25T04:15:41Z"
  name: sample-sc-token-wmdvw
  namespace: default
  resourceVersion: "10097"
  uid: b31a06c6-e26e-411d-a3ff-614589691bc4
type: kubernetes.io/service-account-token
```

其中 {data.token} 就會是我們的用戶 token 的 base64 編碼，之後我們配置 kubeconfig 的時候將會用到它；

## 創建角色

比如我們想創建一個只可以查看集群 `deployments`，`services`，`pods` 相關的角色，應該使用如下配置:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
## 这里也可以使用 Role
kind: ClusterRole
metadata:
  name: mofang-viewer-role
  labels:
    from: mofang
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/status
  - pods/log
  - services
  - services/status
  - endpoints
  - endpoints/status
  - deployments
  verbs:
  - get
  - list
  - watch
```

## 創建角色綁定

```yaml
apiVersion: rbac.authorization.k8s.io/v1
## 这里也可以使用 RoleBinding
kind: ClusterRoleBinding
metadata:
  name: sample-role-binding
  labels:
    from: mofang
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: mofang-viewer-role
subjects:
- kind: ServiceAccount
  name: sample-sc
  namespace: default
```

## 使用多個Service Account

每個 namespace 中都有一個默認的叫做 `default` 的 service account 資源。

您可以使用以下命令列出 namespace 下的所有 serviceAccount 資源。

```bash
$ kubectl get serviceaccounts

NAME        SECRETS   AGE
default     1         3h19m
```

您可以像這樣創建一個 ServiceAccount 對象：

```
$ cat > /tmp/serviceaccount.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
EOF

$ kubectl create -f /tmp/serviceaccount.yaml
serviceaccount "build-robot" created
```

如果您看到如下的 service account 對象的完整輸出信息：

```
$ kubectl get serviceaccounts/build-robot -o yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-06-25T04:26:12Z"
  name: build-robot
  namespace: default
  resourceVersion: "10625"
  uid: e9d34d79-c412-410c-baa5-f985096e105e
secrets:
- name: build-robot-token-4mmqj
```

然後您將看到有一個 token 已經被自動創建，並被 service account 引用。

```bash
$ kubectl get secret build-robot-token-4mmqj -o yaml
```

結果:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-06-25T04:26:12Z"
  name: build-robot
  namespace: default
  resourceVersion: "10625"
  uid: e9d34d79-c412-410c-baa5-f985096e105e
secrets:
- name: build-robot-token-4mmqj
$ kubectl get secret build-robot-token-4mmqj
NAME                      TYPE                                  DATA   AGE
build-robot-token-4mmqj   kubernetes.io/service-account-token   3      3h58m
$ kubectl get secret build-robot-token-4mmqj -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCakNDQWU2Z0F3SUJBZ0lCQVRBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwdGFXNXAKYTNWaVpVTkJNQjRYRFRJeU1EWXhNakEyTVRReE5Gb1hEVE15TURZeE1EQTJNVFF4TkZvd0ZURVRNQkVHQTFVRQpBeE1LYldsdWFXdDFZbVZEUVRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTUx1ClJibkdZeDhWMWVjaVdwSkloaVBiREtJTTVlbTBteWwxQk1EaUsvS09Lb3BIQm02QjBCT1FDZ2h4bVNNQlYwZWIKSFFDUGxWQWdseE5zTWtLMUJNZzZDMTB3bkZXOUtGTWJoOTRFcHdRWTZSWTRSemlxcHdnMUhHTEQ1UzJKcmd2agpPNTIrb3JnNzVvR3NkUkwrOTV1aEc2aEc5NTI0emtFcmxkNllHMUttV2JsVXptRFc4eXVsdUVQaHlvZGNqR0g0CldVdEhISE1nclQyTytnNml4YmN0aTVxRlAydzVSaGJvVDZtUjRvWUM4eTFtZld4MFppTjZOQ1NGMmZZZ3NEbDkKYUliUVovR1EzNnU5VW9WY3U1bWdxN21sdlI0SWFZNSs0dURWOHFWQWR1Y0QzcXJqK1JZUnllMmpqNXJEU0VRZwpYa3lORS9xZzkyaUc5WTlmbHo4Q0F3RUFBYU5oTUY4d0RnWURWUjBQQVFIL0JBUURBZ0trTUIwR0ExVWRKUVFXCk1CUUdDQ3NHQVFVRkJ3TUNCZ2dyQmdFRkJRY0RBVEFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQjBHQTFVZERnUVcKQkJSd1U0YU9BYTdqVDFlUHNERERNYkJlTWR0L2xqQU5CZ2txaGtpRzl3MEJBUXNGQUFPQ0FRRUFWanplV1FuTwplNytTc1AxRXl1ZURTS0w0cHowQnNtSlFZc3ZBZkROZEphTDBScStJc0FMYmhPWk15Nk4wUnUzdmxrZ1YrY3ZYCkIwSU9zRCsvdlN2bUFtYWNwclJjM3ZrS1cyak1lZy9xR3JWTy8vK1Y5N25Fb1IvbGRtTDB4NEl0bmFaNW1aWHkKOTNkWUVZY0tkb2pqTXY1S0dKbjlvcmhRS2Z0SzlYUk1zZ3kxUW5XVGJMbUEyeENMUGtuQzhoV0U1QS85bXlFRgo3UTVtaFFSWS85akQ2ZVJFeERBbytuKzRlVHlPd3loa3pXenEyMWVpNmJNbUt0SVlUdlc1SFNOSXo3NzNoV3grCjNjaUZlSzZTcHg0c0RpckdYenFjaEk4dHpTQ3NNS1o1ODAwSWp1T1RNb0VCM01WN0Rhalk0eE1MMW1nZlBta1EKRmtxZWQwcGhqNVRsUEE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  namespace: ZGVmYXVsdA==
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltUk9hbkJKYmpGd2JVbElRbHByU1VKb2VrdExTV2xoZFRBMk9IWkhXbTFWVlVsNWF6SnRRMGgyWjBFaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbUoxYVd4a0xYSnZZbTkwTFhSdmEyVnVMVFJ0YlhGcUlpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVibUZ0WlNJNkltSjFhV3hrTFhKdlltOTBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpYSjJhV05sTFdGalkyOTFiblF1ZFdsa0lqb2laVGxrTXpSa056a3RZelF4TWkwME1UQmpMV0poWVRVdFpqazROVEE1Tm1VeE1EVmxJaXdpYzNWaUlqb2ljM2x6ZEdWdE9uTmxjblpwWTJWaFkyTnZkVzUwT21SbFptRjFiSFE2WW5WcGJHUXRjbTlpYjNRaWZRLjVMTWhjR0hfMk1tTVhHSHpIQkhiZmhqN21rT0RtRV81UjE2SWE2ZHhzeVNvRURNZW5lSVc0R212MWxFTDlwTjlyZE9HR1lwRWNqR2dxZmc0Q0hESFlCYVhxLVdwMy05dmMwci1JQjEyY0xBWVhoMllXN2JwY0F5eVJOdUhSZzN3V1ZPWm9nQ0JoZmRISS1uaWNHYnF3Y3lFSzVvNllUOWNRajJDcFAxeFVYN1R0U2dTbWc0TDNscXBoaGhYYVo1RFVtT2FldlE2REhGZEVBYXpsOGN6WURRb01yTi1GekpUVEgwZzBJOWJ2YnAtZDFUa29hWnNXaG15RUROVV9jeTRVaEdrYl9BNGJLckVKQ2xkMG8yM1BJVVpPWFBBdEQ3MFRZN2VQcVpycllaRWRWUmN5WWRzb2pPSVlobUZXWEVTRWg5dG5KTFdOZ3ZoaXFwd2U5dEVFQQ==
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: build-robot
    kubernetes.io/service-account.uid: e9d34d79-c412-410c-baa5-f985096e105e
  creationTimestamp: "2022-06-25T04:26:12Z"
  name: build-robot-token-4mmqj
  namespace: default
  resourceVersion: "10624"
  uid: f22e7413-eda5-4b0f-bbbd-bb353588ff01
type: kubernetes.io/service-account-token
```

設置非默認的 service account，只需要在 pod 的 `spec.serviceAccountName` 字段中將 `name` 設置為您想要用的 service account 名字即可。

在 pod 創建之初 service account 就必須已經存在，否則創建將被拒絕。

您不能更新已創建的 pod 的 service account。

您可以清理 service account，如下所示：

```bash
$ kubectl delete serviceaccount/build-robot
```

## 手動創建 service account 的 API token

假設我們已經有了一個如上文提到的名為 ”build-robot“ 的 service account，我們手動創建一個新的 secret。

```bash
$ cat > /tmp/build-robot-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations: 
    kubernetes.io/service-account.name: build-robot
type: kubernetes.io/service-account-token
EOF

$ kubectl create -f /tmp/build-robot-secret.yaml

secret "build-robot-secret" created
```

現在您可以確認下新創建的 secret 取代了 "build-robot" 這個 service account 原來的 API token。

所有已不存在的 service account 的 token 將被 token controller 清理掉。

```bash
$ kubectl describe secrets/build-robot-secret
Name:         build-robot-secret
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: build-robot
              kubernetes.io/service-account.uid: e9d34d79-c412-410c-baa5-f985096e105e

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1111 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImROanBJbjFwbUlIQlprSUJoektLSWlhdTA2OHZHWm1VVUl5azJtQ0h2Z0EifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImJ1aWxkLXJvYm90LXNlY3JldCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJidWlsZC1yb2JvdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImU5ZDM0ZDc5LWM0MTItNDEwYy1iYWE1LWY5ODUwOTZlMTA1ZSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmJ1aWxkLXJvYm90In0.WhfmWnNQaTUxALMz5vGJmeL-xCMKqxIjWAWnc4OH6ACiRdwwG_TgiphT4_1XlmuUDKSd2hS-yYdLSnWADrNyLBj-5GFcClI4CN8i6VDPoM2FRRlDPWJ0Yr56Alj1-tK_mbIlJck5GyAGrN32Ju2jDxlb0U_GSh5qf85-BijIC_c3AyFxO1W4jbkHNbXpvsUsu71xesAJamlygQwOwxXyL-UYducwFEHebwk4_o4gSd3_m5YwUKEFSfNlrUiCm-Fy91rOCRshDPRbwwUTaW3jCBZIMK-GjgSRTvTJE1dZb5dlieqhPevSBXxw85z8gIsfJ99zuHN2no9avdIImqyxBA
```