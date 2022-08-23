# Templates

ApplicationSet `spec` 的模板字段用於生成 Argo CD `Application` 資源。

## Template fields

通過將來自生成器的參數與模板的字段（通過 `{{values}}`）組合來創建 Argo CD 應用程序，並從中生成具體的應用程序資源並將其應用於集群。

這是來自集群生成器的模板 `subfields`：

```yaml
# (...)
 template:
   metadata:
     name: '{{cluster}}-guestbook'
   spec:
     source:
       repoURL: https://github.com/infra-team/cluster-deployments.git
       targetRevision: HEAD
       path: guestbook/{{cluster}}
     destination:
       server: '{{url}}'
       namespace: guestbook
```

模板 `subfields` 直接對應於 [Argo CD `Application` 資源的規範](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications)：

- `project` 是指正在使用的 [Argo CD 項目](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)（這裡可以使用默認值來利用默認的 Argo CD 項目）
- `source` 定義從哪個 Git 存儲庫中提取所需的應用程序清單
    - **repoURL**：存儲庫的 URL（例如 https://github.com/argoproj/argocd-example-apps.git）
    - **targetRevision**：存儲庫的修訂版（標籤/分支/提交）（例如 HEAD）
    - **path**：Kubernetes 清單（和/或 Helm、Kustomize、Jsonnet 資源）所在的存儲庫中的路徑
- `destination`：定義要部署到哪個 Kubernetes 集群/命名空間
    - **name**：要部署到的集群的名稱（在 Argo CD 中）
    - **server**：集群的 API Server URL（例如：`https://kubernetes.default.svc`）
    - **namespace**：從 `source` 部署清單的目標命名空間（示例：my-app-namespace）

!!! info
    - 引用的集群必須已經在 Argo CD 中定義，ApplicationSet 控制器才能使用它們
    - 只能指定 `name` 或 `server` 之一：如果兩者都指定，則返回錯誤。

模板的元數據字段也可用於設置應用程序名稱，或為應用程序添加標籤或註釋。

雖然 ApplicationSet 規範提供了一種基本的模板形式，但它並不打算取代 Kustomize、Helm 或 Jsonnet 等工具的成熟配置管理功能。

## Generator templates

除了在 `ApplicationSet` 資源的 `.spec.template` 中指定模板外，還可以在生成器中指定模板。這對於覆蓋 `spec`-level 模板的值很有用。

生成器的 `template` 字段優先於模板 `spec` 字段：

- 如果兩個模板包含相同的字段，則將使用生成器的字段值。
- 如果這些模板的字段中只有一個具有值，則將使用該值。

因此，生成器模板可以被認為是針對外部 `spec`-level 模板字段的補丁。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - list:
      elements:
        - cluster: engineering-dev
          url: https://kubernetes.default.svc
      template:
        metadata: {}
        spec:
          project: "default"
          source:
            revision: HEAD
            repoURL: https://github.com/argoproj/applicationset.git
            # New path value is generated here:
            path: 'examples/template-override/{{cluster}}-override'
          destination: {}

  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: "default"
      source:
        repoURL: https://github.com/argoproj/applicationset.git
        targetRevision: HEAD
        # This 'default' value is not used: it is is replaced by the generator's template path, above
        path: examples/template-override/default
      destination:
        server: '{{url}}'
        namespace: guestbook
```

（完整的例子可以在[這裡](https://github.com/argoproj/applicationset/tree/master/examples/template-override)找到。）

在本例中，`ApplicationSet` 控制器將使用 List 生成器生成的路徑而不是 `.spec.template` 中定義的路徑值來生成 `Application` 資源。