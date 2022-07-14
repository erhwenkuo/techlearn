# Kubernetes API 基礎

原文: https://iximiuz.com/en/posts/kubernetes-api-structure-and-terminology/

這是關於如何從代碼中使用 Kubernetes API 的系列文章中的第一篇。 Kubernetes API 比一堆 HTTP 端點組合在一起要先進一點。因此，在嘗試從代碼中訪問它之前，了解 Kubernetes API 結構並熟悉術語至關重要。否則，嘗試將非常痛苦——官方的 Go 客戶端帶有太多的花里胡哨，試圖將你的頭腦圍繞在客戶端和 API 概念上，可能很快就會讓你不知所措。

Kubernetes API 非常龐大——它有數百個端點。幸運的是，它非常一致，因此只需了解有限數量的想法，然後將這些知識外推到 API 的其餘部分。在這篇文章中，我將嘗試觸及我發現的最基本的概念。我傾向於簡單和易消化，而不是學術上的正確性和材料的完整性。和往常一樣，我只是分享我對事物的理解和思考這個話題的方式——所以，它不是 API 手冊，而是個人學習經驗的記錄。

![](./assets/k8s-api-intro.png)

## Resources and Verbs

由於它是一個 RESTful 領域，我們將根據`resource`（鬆散地，某種結構的對象）和`verb`（對這些對象的操作）進行操作。

在討論`resources`時，將 `a resource as a certain kind of objects` 與 `a resource as a particular instance of some kind` 區分開來是很重要的。因此，Kubernetes API 端點被正式命名為資源類型 (resource types)，以避免與資源實例 (resource instances) 產生歧義。然而，對一般應用程式的開發者來，端點(endpoint)通常被稱為資源(resource)，因此這個詞的實際含義得從上下文中來得出。

出於可擴展性的原因，`resource types`　被組織成 `API groups`，並且這些　API　群組彼此獨立地進行版本控制：

```bash
$ kubectl api-resources

NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume
pods                              po           v1                                     true         Pod
podtemplates                                   v1                                     true         PodTemplate
replicationcontrollers            rc           v1                                     true         ReplicationController
resourcequotas                    quota        v1                                     true         ResourceQuota
secrets                                        v1                                     true         Secret
serviceaccounts                   sa           v1                                     true         ServiceAccount
services                          svc          v1                                     true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io/v1              false        APIService
controllerrevisions                            apps/v1                                true         ControllerRevision
daemonsets                        ds           apps/v1                                true         DaemonSet
deployments                       deploy       apps/v1                                true         Deployment
replicasets                       rs           apps/v1                                true         ReplicaSet
statefulsets                      sts          apps/v1                                true         StatefulSet
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io/v1                true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling/v2                         true         HorizontalPodAutoscaler
cronjobs                          cj           batch/v1                               true         CronJob
jobs                                           batch/v1                               true         Job
certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
leases                                         coordination.k8s.io/v1                 true         Lease
endpointslices                                 discovery.k8s.io/v1                    true         EndpointSlice
events                            ev           events.k8s.io/v1                       true         Event
flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta2   false        FlowSchema
prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta2   false        PriorityLevelConfiguration
ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
ingresses                         ing          networking.k8s.io/v1                   true         Ingress
networkpolicies                   netpol       networking.k8s.io/v1                   true         NetworkPolicy
runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
poddisruptionbudgets              pdb          policy/v1                              true         PodDisruptionBudget
podsecuritypolicies               psp          policy/v1beta1                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io/v1           true         RoleBinding
roles                                          rbac.authorization.k8s.io/v1           true         Role
priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
csinodes                                       storage.k8s.io/v1                      false        CSINode
csistoragecapacities                           storage.k8s.io/v1beta1                 true         CSIStorageCapacity
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment

```

如果你好奇 `kubectl api-resources` 命令是如何構建這樣一個支持的資源列表的，這裡有一個很好的技巧，它顯示了任何 kubectl 命令進行了哪些 API 調用：

```bash
$ kubectl api-resources -v 6  # -v 6 means "extra verbose logging"

...
I0108 ... GET https://192.168.58.2:8443/api?timeout=32s 200 OK in 10 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis?timeout=32s 200 OK in 1 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis/apiregistration.k8s.io/v1?timeout=32s 200 OK in 7 milliseconds
I0108 ... GET https://192.168.58.2:8443/api/v1?timeout=32s 200 OK in 13 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis/authentication.k8s.io/v1?timeout=32s 200 OK in 13 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis/events.k8s.io/v1?timeout=32s 200 OK in 15 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis/apps/v1?timeout=32s 200 OK in 14 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis/autoscaling/v2beta1?timeout=32s 200 OK in 16 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis/policy/v1beta1?timeout=32s 200 OK in 14 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis/scheduling.k8s.io/v1?timeout=32s 200 OK in 14 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis/batch/v1?timeout=32s 200 OK in 13 milliseconds
I0108 ... GET https://192.168.58.2:8443/apis/batch/v1beta1?timeout=32s 200 OK in 43 milliseconds
...
```

其實　ubernetes　的　API　都具備了相當豐富的 meta data 來陳述這個　Kubernetes API 基本資料。顯然，您可以通過讀取（或創建）其他資源來列出現有的（甚至註冊新的資源類型）！例如，上面的列表是通過調用特殊的 `/api` 和 `/apis/<group-name>` 資源獲得的。

!!! info
    `/api` 端點已經是最初　Kubernetes　定義與使用的，僅用於核心資源（pod、secrets、configmaps 等）。一個更現代和通用的 `/apis/<group-name>` 端點用於其餘資源，包括用戶定義的自定義資源。

您可以使用 curl 等標準 HTTP 客戶端輕鬆調用上述資源（即 API 端點）並檢查返回的資源（即 JSON 對象）：

```bash
# Make Kubernetes API available on localhost:8080
# to bypass the auth step in subsequent queries:
$ kubectl proxy --port=8080 &

# List all known API paths
$ curl http://localhost:8080/
# List known versions of the `core` group
$ curl http://localhost:8080/api
# List known resources of the `core/v1` group
$ curl http://localhost:8080/api/v1
# Get a particular Pod resource
$ curl http://localhost:8080/api/v1/namespaces/default/pods/sleep-7c7db887d8-dkkcg

# List known groups (all but `core`)
$ curl http://localhost:8080/apis
# List known versions of the `apps` group 
$ curl http://localhost:8080/apis/apps
# List known resources of the `apps/v1` group
$ curl http://localhost:8080/apis/apps/v1
# Get a particular Deployment resource
$ curl http://localhost:8080/apis/apps/v1/namespaces/default/deployments/sleep

```

!!! info
    有一種更簡單的方法來檢查 Kubernetes API：`kubectl get --raw /SOME/API/PATH`。然而，上面的練習是為了表明 Kubernetes API 並不神奇。擁有一個通用型的 HTTP 客戶端或工具就足以開始使用它了。

說到　`verbs` (即對資源的操作)，所有標準的 CRUD 操作及其傳統映射到 HTTP 方法都在那裡。此外，還支持資源的`patching`（選擇性字段修改）和 `watching`（流式集合的讀取）。在　Kubernetes 官方開發指南中定義了詳細的說明: [sig-architecture/api-conventions.md](https://github.com/kubernetes/community/blob/7f3f3205448a8acfdff4f1ddad81364709ae9b71/contributors/devel/sig-architecture/api-conventions.md#verbs-on-resources)：

```
## Verbs on Resources

API resources should use the traditional REST pattern:

GET /<resourceNamePlural> - Retrieve a list of type <resourceName>, e.g. GET /pods returns a list of Pods.
POST /<resourceNamePlural> - Create a new resource from the JSON object provided by the client.
GET /<resourceNamePlural>/<name> - Retrieves a single resource with the given name, e.g. GET /pods/first returns a Pod named 'first'. Should be constant time, and the resource should be bounded in size.
DELETE /<resourceNamePlural>/<name> - Delete the single resource with the given name. DeleteOptions may specify gracePeriodSeconds, the optional duration in seconds before the object should be deleted. Individual kinds may declare fields which provide a default grace period, and different kinds may have differing kind-wide default grace periods. A user provided grace period overrides a default grace period, including the zero grace period ("now").
DELETE /<resourceNamePlural> - Deletes a list of type <resourceName>, e.g. DELETE /pods a list of Pods.
PUT /<resourceNamePlural>/<name> - Update or create the resource with the given name with the JSON object provided by the client.
PATCH /<resourceNamePlural>/<name> - Selectively modify the specified fields of the resource. See more information below.
GET /<resourceNamePlural>?watch=true - Receive a stream of JSON objects corresponding to changes made to any resource of the given kind over time.

## PATCH operations

The API supports three different PATCH operations, determined by their corresponding Content-Type header:

JSON Patch, Content-Type: application/json-patch+json
As defined in RFC6902, a JSON Patch is a sequence of operations that are executed on the resource, e.g. {"op": "add", "path": "/a/b/c", "value": [ "foo", "bar" ]}. For more details on how to use JSON Patch, see the RFC.
Merge Patch, Content-Type: application/merge-patch+json
As defined in RFC7386, a Merge Patch is essentially a partial representation of the resource. The submitted JSON is "merged" with the current resource to create a new one, then the new one is saved. For more details on how to use Merge Patch, see the RFC.
Strategic Merge Patch, Content-Type: application/strategic-merge-patch+json
Strategic Merge Patch is a custom implementation of Merge Patch. For a detailed explanation of how it works and why it needed to be introduced, see here.
```

## Kinds aka Object Schemas

`kind`(類型) 這個詞會時不時地出現在這里和那裡。例如，在 `kubectl api-resources` 輸出中，您可以看到 `persistentvolumes` 資源具有相應的 `PersistentVolume` 類型。

很長一段時間以來，我與 Kubernetes 的交互僅限於使用 `kubectl apply` 盲目地為其提供清單。這讓我覺得 kind 總是包含一個資源的 CamelCase 名稱，比如 `Pod`、`Service`、`Deployment` 等。

```bash hl_lines="3"
$ cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    ...
EOF
```

但實際上，就算不是資源的 Kubernetes 數據結構也可以有`Kind`的宣告：

```yaml hl_lines="2"
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata

```

甚至 Kubernetes 對象（即持久實體）的資源也有`Kind`：

```bash
$ kubectl get --raw /api | jq
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.49.2:8443"
    }
  ]
}
```

那麼，什麼是`kind`？

!!! info
    "All resource types have a concrete representation which is called a kind" - Kubernetes API reference. 嗯，這個解釋不是特別有用! 🤔

事實證明，在 Kubernetes 中，一種`Kind`是 `object schema`。就像您通常使用 JSON schema 那樣。換句話說，`Kind`是指特定的數據結構，即 attributes 和 properties 的某種組合。

根據 [sig-architecture/api-conventions.md](https://github.com/kubernetes/community/blob/7f3f3205448a8acfdff4f1ddad81364709ae9b71/contributors/devel/sig-architecture/api-conventions.md#types-kinds)，`Kind` 分為三類：

- `Objects` (Pod, Service, etc) - persistent entities in the system.
- `Lists` - (PodList, APIResourceList, etc) - collections of resources of one or more kinds.
- `Simple` - specific actions on objects (status, scale, etc.) or non-persistent auxiliary entities (ListOptions, Policy, etc).

Kubernetes 中使用的大多數物件(object)，包括 API 返回的所有 JSON 物件，都有 `kind` 字段。它允許客戶端和服務器在通過網絡發送它們或將它們存儲在磁盤上之前正確地序列化和反序列化這些物件。

## Kubernetes Objects

就像 `resource` 一樣，Kubernetes 用語中的 `object`一詞是也是搞的大家暈頭轉向的。從廣義上講，`Objects` 可以指任何數據結構——資源類型的實例（如 APIGroup）、配置（如審計策略）或持久實體（如 Pod）。但是，在本節中，我將在狹義的、明確定義的意義上討論 `object`。所以，我將使用大寫的詞 `Object` 來代替。

大多數 Kubernetes API 資源代表 `Object`。與僅要求 `kind` 字段的其他形式的資源不同，Objects 必須定義更多字段：

- `kind` - a string that identifies the schema this object should have
- `apiVersion` - a string that identifies the version of the schema the object should have
- `metadata.namespace` - a string with the namespace (defaults to "default")
- `metadata.name` - a string that uniquely identifies this object within the current namespace
- `metadata.uid` - a unique in time and space value used to distinguish between objects with the same name that have been deleted and recreated.

此外，`metadata` 也可能包括標籤和註釋字段，以及一些版本控制和時間戳信息。

示例 - Pod 對象（截斷輸出）：

```bash
$ kubectl get --raw /api/v1/namespaces/default/pods/sleep-7c7db887d8-dkkcg | jq
{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
        "namespace": "default",
        "name": "sleep-7c7db887d8-dkkcg",        
        "uid": "32bf410a-0009-484e-adac-21179ec28f0f",
        "labels": {
            "app": "sleep",
            "pod-template-hash": "7c7db887d8"
        },        
        "creationTimestamp": "2022-01-08T18:10:04Z",
        "resourceVersion": "465766"        
    },
    "spec": { ... },
    "status": { ... }
}
```

如上例所示，Kubernetes 對象通常具有 `spec`（期望狀態）和 `status`（實際狀態）字段。但情況並非總是如此。將上面的輸出與下面的 ConfigMap 對象進行比較：

```bash
$ kubectl get --raw /api/v1/namespaces/default/configmaps/informer-dynamic-simple-wzgmx | jq
{
    "apiVersion": "v1",
    "kind": "ConfigMap",
    "data": {
        "foo": "bar"
    },    
    "metadata": {
        "namespace": "default",
        "name": "informer-dynamic-simple-wzgmx",
        "uid": "74471398-0244-4686-b490-7007f6246a63",
        "creationTimestamp": "2022-01-06T21:45:04Z",
        "generateName": "informer-dynamic-simple-",
        "resourceVersion": "418185"        
    }
}
```

從代碼中處理 Kubernetes API 涉及到大量的 `Object` 操作，因此必須對常見的對象結構有紮實的理解。 `kubectl explain` 命令可以幫助您。它最酷的部分是它不僅可以在資源上調用，還可以在嵌套字段上調用：

```bash
$ kubectl explain deployment.spec.template
KIND:     Deployment
VERSION:  apps/v1

RESOURCE: template <Object>

DESCRIPTION:
     Template describes the pods that will be created.

     PodTemplateSpec describes the data a pod should have when created from a
     template

FIELDS:
   metadata <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec <Object>
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
```

## Summarizing

Kubernetes 用語中的 `resource` 可以指資源類型和資源實例。資源類型被組織成 API groups，並且 API groups是版本化的。每個資源表示都遵循由其種類定義的特定模式。雖然每個資源都遵循由其種類定義的具體結構，但並非每個資源都代表一個 Kubernetes 對象。對像是表示意圖記錄的持久實體。不同種類的對象具有不同的結構，但所有對像都帶有共同的元數據屬性，例如命名空間、名稱、uid 或 creationTimestamp。

![](./assets/resource-types-kinds-objects-2000-opt.png)

## What's next?

對 Kubernetes API 的理論部分有更好的了解嗎？ 太好了！　接下來我們將用一個簡單的 HTTP 客戶端調用它(事實證明這不是一件容易的事，有很多坑要跳)。

當理論和實踐相提並論時，我建議看一下 [k8s.io/api](https://github.com/kubernetes/api) 和 [k8s.io/apimachinery](https://github.com/kubernetes/apimachinery) 模塊 - 這是[官方 Go 客戶端](https://github.com/kubernetes/client-go/)的兩個主要依賴項。 [k8s.io/api](https://github.com/kubernetes/api)為 Kubernetes 對象定義的 Go 結構，而 [k8s.io/apimachinery](https://github.com/kubernetes/apimachinery)帶來了較低級別的構建塊和常見的 API 功能，如序列化、類型轉換或錯誤處理。[這是我對這兩個模塊的圖解概述](https://iximiuz.com/en/posts/kubernetes-api-go-types-and-common-machinery/)。

Talk is cheap, show me the code? 大家可看我在 GitHub 上收集的 [Kubernetes client-go](https://github.com/iximiuz/client-go-examples) 示例😉

**Resources**

- [Kubernetes Documentation - API Overview](https://kubernetes.io/docs/reference/using-api/)
- [Kubernetes Documentation - API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)
- [Kubernetes Documentation- Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
- [SIG Architecture - API Conventions](https://github.com/kubernetes/community/blob/7f3f3205448a8acfdff4f1ddad81364709ae9b71/contributors/devel/sig-architecture/api-conventions.md)
- [Building stuff with the Kubernetes API — Exploring API objects](https://medium.com/programming-kubernetes/building-stuff-with-the-kubernetes-api-1-cc50a3642)
- 