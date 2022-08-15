# 配置最佳實踐

本文檔重點介紹並整合了整個 Kubernetes 用戶指南、入門文檔和示例中介紹的配置最佳實踐。

## 一般配置提示

- 定義配置時，請指定最新的穩定 API 版本。
- 在推送到集群之前，配置文件應存儲在版本控制中。這允許你在必要時快速回滾配置更改。它還有助於集群重新創建和恢復。
- 使用 YAML 而不是 JSON 編寫配置文件。雖然這些格式幾乎可以在所有場景中互換使用，但 YAML 往往更加用戶友好。
- 只要有意義，就將相關對象分組到一個文件中。一個文件通常比幾個文件更容易管理。請參閱 guestbook-all-in-one.yaml 文件作為此語法的示例。
- 另請注意，可以在目錄上調用許多 kubectl 命令。例如，你可以在配置文件的目錄中調用 kubectl apply。
- 除非必要，否則不指定默認值：簡單的最小配置會降低錯誤的可能性。
- 將對象描述放在註釋中，以便更好地進行內省(introspection)。

## “Naked” Pods 与 ReplicaSet，Deployment 和 Jobs 

- 如果可能，不要使用獨立的 Pods（即，未綁定到 ReplicaSet 或 Deployment 的 Pod）。如果節點發生故障，將不會重新調度獨立的 Pods。

Deployment 既可以創建一個 ReplicaSet 來確保預期個數的 Pod 始終可用，也可以指定替換 Pod 的策略（例如 RollingUpdate）。除了一些顯式的 `restartPolicy: Never` 場景外，Deployment 通常比直接創建 Pod 要好得多。 Job 也可能是合適的選擇。

## 服務

- 在創建相應的後端工作負載（Deployment 或 ReplicaSet），以及在需要訪問它的任何工作負載之前創建 [服務](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)。當 Kubernetes 啟動容器時，它提供指向啟動容器時正在運行的所有服務的環境變量。例如，如果存在名為 foo 的服務，則所有容器將在其初始環境中獲得以下變量。

    ```bash
    FOO_SERVICE_HOST=<the host the Service is running on>
    FOO_SERVICE_PORT=<the port the Service is running on>
    ```

    這確實意味著在順序上的要求 - 必須在 Pod 本身被創建之前創建 Pod 想要訪問的任何 Service， 否則將環境變量不會生效。 DNS 沒有此限制。

- 一個非必要（儘管強烈推薦）的[集群插件](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/) 是 DNS 服務器。 DNS 服務器為新的 Services 監視 Kubernetes API，並為每個創建一組 DNS 記錄。如果在整個集群中啟用了 DNS，則所有 Pods 應該能夠自動對 Services 進行名稱解析。

- 除非絕對必要，否則不要為 Pod 指定 hostPort。將 Pod 綁定到 hostPort 時，它會限制 Pod 可以調度的位置數，因為每個 <hostIP, hostPort, protocol> 組合必須是唯一的。如果你沒有明確指定 hostIP 和 protocol，Kubernetes 將使用 0.0.0.0 作為默認 hostIP 和 TCP 作為默認 protocol。

    如果你只需要訪問端口以進行調試，則可以使用 [apiserver proxy](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/access-cluster/#manually-constructing-apiserver-proxy-urls) 或 [`kubectl port-forward`](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)。

    如果你明確需要在節點上公開 Pod 的端口，請在使用 hostPort 之前考慮使用 [NodePort](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#type-nodeport) 服務。

## 使用標籤

- 定義並使用標籤來識別應用程序 或 Deployment 的 **語義屬性**，例如`{ app.kubernetes.io/name: MyApp, tier: frontend, phase: test, deployment: v3 }`。你可以使用這些標籤為其他資源選擇合適的 Pod； 例如，一個選擇所有 `tier: frontend Pod` 的服務，或者 `app.kubernetes.io/name: MyApp` 的所有 `phase: test` 組件。有關此方法的示例，請參閱 [guestbook](https://github.com/kubernetes/examples/tree/master/guestbook/) 。

    通過從選擇器中省略特定發行版的標籤，可以使服務跨越多個 Deployment。當你需要不停機的情況下更新正在運行的服務，可以使用Deployment。

    Deployment 描述了對象的期望狀態，並且如果對該規範的更改被成功應用， 則 Deployment 控制器以受控速率將實際狀態改變為期望狀態。

- 對於常見場景，應使用 [Kubernetes 通用[標籤](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/common-labels/)。這些標準化的標籤豐富了對象的元數據，使得包括 `kubectl` 和 [儀表板（Dashboard）](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/web-ui-dashboard) 這些工具能夠以可互操作的方式工作。

- 你可以操縱標籤進行調試。由於 Kubernetes 控制器（例如 ReplicaSet）和服務使用選擇器標籤來匹配 Pod， 從 Pod 中刪除相關標籤將阻止其被控制器考慮或由服務提供服務流量。如果刪除現有 Pod 的標籤，其控制器將創建一個新的 Pod 來取代它。這是在"隔離"環境中調試先前"活躍"的 Pod 的有用方法。要以交互方式刪除或添加標籤，請使用 [`kubectl label`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#label)。

## 使用 kubectl

- 使用 `kubectl apply -f <directory>`。它在 `<directory>` 中的所有 `.yaml、.yml` 和 `.json` 文件中查找 Kubernetes 配置，並將其傳遞給 `apply`。
- 使用標籤選擇器進行 get 和 delete 操作，而不是特定的對象名稱。
- 請參閱[標籤選擇器](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/#label-selectors)和 [有效使用標籤](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/manage-deployment/#using-labels-effectively)部分。
- 使用`kubectl run`和`kubectl expose`來快速創建單容器部署和服務。有關示例，請參閱[使用服務訪問集群中的應用程序](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/service-access-application-cluster/)。