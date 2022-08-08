# 使用 Kustomize 對 Kubernetes 對象進行聲明式管理

![](./assets/kustomize.png)

[Kustomize](https://kustomize.io/) 是一個獨立的工具，用來通過 [kustomization 文件](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#kustomization) 定制 Kubernetes 對象。

從 1.14 版本開始，kubectl 也開始支持使用 kustomization 文件來管理 Kubernetes 對象。要查看包含 kustomization 文件的目錄中的資源，執行下面的命令：

```bash
kubectl kustomize <kustomization_directory>
```

要應用這些資源，使用參數 `--kustomize` 或 `-k` 標誌來執行 `kubectl apply`：

```bash
kubectl apply -k <kustomization_directory>
```

## 準備開始

- 安裝 [kubectl](https://kubernetes.io/zh-cn/docs/tasks/tools/)

你必須擁有一個 Kubernetes 的集群，同時你的 Kubernetes 集群必須帶有 kubectl 命令行工具。如果你還沒有集群，你可以通過 [K3D](../../01-getting-started/learning-env/k3d/k3s-kubernetes-cluster-setup-with-k3d.md) 構建一個你自己的集群。

## Kustomize 概述

Kustomize 是一個用來定制 Kubernetes 應用配置的工具。它提供以下功能特性來管理 **應用配置文件**：

- 從其他來源生成資源物件
- 為資源物件設置貫穿性（Cross-Cutting）字段
- 組織和定制資源物件集合

### 生成资源對象

ConfigMap 和 Secret 包含其他 Kubernetes 對象（如 Pod）所需要的配置或敏感數據。 ConfigMap 或 Secret 中數據的來源往往是集群外部，例如某個 `.properties` 文件或者 SSH 密鑰文件。 Kustomize 提供 `secretGenerator` 和 `configMapGenerator`，可以基於文件或字面 值來生成 Secret 和 ConfigMap。

#### configMapGenerator

要基於文件來生成 ConfigMap，可以在 `configMapGenerator` 的 `files` 列表中添加表項。下面是一個根據 `.properties` 文件中的數據條目來生成 ConfigMap 的示例：

```bash
# 生成一个  application.properties 文件
cat <<EOF >application.properties
FOO=Bar
EOF

cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
EOF
```

所生成的 ConfigMap 可以使用下面的命令来检查：

```bash
kubectl kustomize ./
```

所生成的 ConfigMap 為：

```yaml
apiVersion: v1
data:
  application.properties: |
        FOO=Bar
kind: ConfigMap
metadata:
  name: example-configmap-1-8mbdf7882g
```

要從 env 文件生成 ConfigMap，請在 `configMapGenerator` 中的 `envs` 列表中添加一個條目。這也可以用於通過省略 `=` 和值來設置本地環境變量的值。

建議謹慎使用本地環境變量填充功能 —— 用補丁覆蓋通常更易於維護。當無法輕鬆預測變量的值時，從環境中設置值可能很有用，例如 git SHA。

下面是一個用來自 `.env` 文件的數據生成 ConfigMap 的例子：

```
# 创建一个 .env 文件
# BAZ 将使用本地环境变量 $BAZ 的取值填充
cat <<EOF >.env
FOO=Bar
BAZ
EOF

cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  envs:
  - .env
EOF
```

可以使用以下命令檢查生成的 ConfigMap：

```bash
BAZ=Qux kubectl kustomize ./
```

生成的 ConfigMap 為：

```yaml
apiVersion: v1
data:
  BAZ: Qux
  FOO: Bar
kind: ConfigMap
metadata:
  name: example-configmap-1-892ghb99c8
```

!!! info
    說明： .env 文件中的每個變量在生成的 ConfigMap 中成為一個單獨的鍵。這與之前的示例不同，前一個示例將一個名為 .properties 的文件（及其所有條目）嵌入到同一個鍵的值中。

ConfigMap 也可基於字面的鍵值對來生成。要基於鍵值對來生成 ConfigMap， 在 `configMapGenerator` 的 `literals` 列表中添加表項。下面是一個例子，展示 如何使用鍵值對中的數據條目來生成 ConfigMap 對象：

```
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-2
  literals:
  - FOO=Bar
EOF
```

可以用下面的命令檢查所生成的 ConfigMap：

```bash
kubectl kustomize ./
```

所生成的 ConfigMap 為：

```yaml
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  name: example-configmap-2-g2hdhfc6tk
```

要在 Deployment 中使用生成的 ConfigMap，使用 configMapGenerator 的名稱對其進行引用。 Kustomize 將自動使用生成的名稱替換該名稱。

這是使用生成的 ConfigMap 的 deployment 示例：

```
# 創建一個 application.properties 文件
cat <<EOF >application.properties
FOO=Bar
EOF

cat <<EOF >deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: example-configmap-1
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
EOF
```

生成 ConfigMap 和 Deployment：

```bash
kubectl kustomize ./
```

生成的 Deployment 將通過名稱引用生成的 ConfigMap：

```
apiVersion: v1
data:
  application.properties: |
        FOO=Bar
kind: ConfigMap
metadata:
  name: example-configmap-1-g4hk9g2ff8
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-app
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: my-app
        name: app
        volumeMounts:
        - mountPath: /config
          name: config
      volumes:
      - configMap:
          name: example-configmap-1-g4hk9g2ff8
        name: config
```

#### secretGenerator

你可以基於文件或者鍵值偶對來生成 Secret。要使用文件內容來生成 Secret， 在 secretGenerator 下面的 files 列表中添加表項。下面是一個根據文件中數據來生成 Secret 對象的示例：

```bash
# 創建一個 password.txt 文件
cat <<EOF >./password.txt
username=admin
password=secret
EOF

cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
EOF
```

所生成的 Secret 如下：

```yaml
apiVersion: v1
data:
  password.txt: dXNlcm5hbWU9YWRtaW4KcGFzc3dvcmQ9c2VjcmV0Cg==
kind: Secret
metadata:
  name: example-secret-1-t2kt65hgtb
type: Opaque
```

要基於鍵值偶對字面值生成 Secret，先要在 secretGenerator 的 literals 列表中添加表項。下面是基於鍵值偶對中數據條目來生成 Secret 的示例：

```bash
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-2
  literals:
  - username=admin
  - password=secret
EOF
```

所生成的 Secret 如下：

```yaml
apiVersion: v1
data:
  password: c2VjcmV0
  username: YWRtaW4=
kind: Secret
metadata:
  name: example-secret-2-t52t6g96d8
type: Opaque
```

與 ConfigMaps 一樣，生成的 Secrets 可以通過引用 secretGenerator 的名稱在部署中使用：

```bash
# 創建一個 password.txt 文件
cat <<EOF >./password.txt
username=admin
password=secret
EOF

cat <<EOF >deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app
        volumeMounts:
        - name: password
          mountPath: /secrets
      volumes:
      - name: password
        secret:
          secretName: example-secret-1
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
EOF
```

#### generatorOptions

所生成的 ConfigMap 和 Secret 都會包含內容哈希值後綴。這是為了確保內容髮生變化時，所生成的是新的 ConfigMap 或 Secret。要禁止自動添加後綴的行為，用戶可以使用 `generatorOptions`。除此以外，為生成的 ConfigMap 和 Secret 指定貫穿性選項也是可以的。


```bash
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-3
  literals:
  - FOO=Bar
generatorOptions:
  disableNameSuffixHash: true
  labels:
    type: generated
  annotations:
    note: generated
EOF
```

運行 `kubectl kustomize ./` 來查看所生成的 ConfigMap：

```
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  annotations:
    note: generated
  labels:
    type: generated
  name: example-configmap-3
```

### 設置貫穿性字段

在項目中為所有 Kubernetes 對象設置貫穿性字段是一種常見操作。貫穿性字段的一些使用場景如下：

- 為所有資源設置相同的名字空間
- 為所有對象添加相同的前綴或後綴
- 為對象添加相同的標籤集合
- 為對象添加相同的註解集合

下面是一個例子：

```yaml
# 創建一個 deployment.yaml
cat <<EOF >./deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
EOF

cat <<EOF >./kustomization.yaml
namespace: my-namespace
namePrefix: dev-
nameSuffix: "-001"
commonLabels:
  app: bingo
commonAnnotations:
  oncallPager: 800-555-1212
resources:
- deployment.yaml
EOF
```

執行 `kubectl kustomize ./` 查看這些字段都被設置到 Deployment 資源上：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    oncallPager: 800-555-1212
  labels:
    app: bingo
  name: dev-nginx-deployment-001
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app: bingo
  template:
    metadata:
      annotations:
        oncallPager: 800-555-1212
      labels:
        app: bingo
    spec:
      containers:
      - image: nginx
        name: nginx
```

### 組織和定制資源

一種常見的做法是在項目中構造資源集合併將其放到同一個文件或目錄中管理。 Kustomize 提供基於不同文件來組織資源並向其應用補丁或者其他定制的能力。

#### 組織

Kustomize 支持組合不同的資源。 `kustomization.yaml` 文件的 `resources` 字段 定義配置中要包含的資源列表。你可以將 `resources` 列表中的路徑設置為資源配置文件 的路徑。下面是由 Deployment 和 Service 構成的 NGINX 應用的示例：

```bash
# 創建 deployment.yaml 文件
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# 創建 service.yaml 文件
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

# 創建 kustomization.yaml 來組織以上兩個資源
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

`kubectl kustomize ./` 所得到的資源中既包含 Deployment 也包含 Service 對象。

#### 定制

補丁文件（Patches）可以用來對資源執行不同的定制。 Kustomize 通過 `patchesStrategicMerge` 和 `patchesJson6902` 支持不同的打補丁 機制。 `patchesStrategicMerge` 的內容是一個文件路徑的列表，其中每個文件都應可解析為 **策略性合併補丁（Strategic Merge Patch）**。補丁文件中的名稱必須與已經加載的資源的名稱匹配。建議構造規模較小的、僅做一件事情的補丁。例如，構造一個補丁來增加 Deployment 的副本個數；構造另外一個補丁來設置內存限制。

```bash
# 創建 deployment.yaml 文件
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# 生成一個補丁 increase_replicas.yaml
cat <<EOF > increase_replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
EOF

# 生成另一個補丁 set_memory.yaml
cat <<EOF > set_memory.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        resources:
          limits:
            memory: 512Mi
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
patchesStrategicMerge:
- increase_replicas.yaml
- set_memory.yaml
EOF
```

執行 `kubectl kustomize ./` 來查看 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 512Mi
```

並非所有資源或者字段都支持策略性合併補丁。為了支持對任何資源的任何字段進行修改， Kustomize 提供通過 `patchesJson6902` 來應用 JSON 補丁 的能力。為了給 JSON 補丁找到正確的資源，需要在 `kustomization.yaml` 文件中指定資源的 組（group）、版本（version）、類別（kind）和名稱（name）。例如，為某 Deployment 對象增加副本個數的操作也可以通過 `patchesJson6902` 來完成：

```bash
# 創建一個 deployment.yaml 文件
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# 創建一個 JSON 補丁文件
cat <<EOF > patch.yaml
- op: replace
  path: /spec/replicas
  value: 3
EOF

# 創建一個 kustomization.yaml
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml

patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-nginx
  path: patch.yaml
EOF
```

執行 `kubectl kustomize ./` 以查看 replicas 字段被更新：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
```

除了補丁之外，Kustomize 還提供定制容器鏡像或者將其他對象的字段值注入到容器 中的能力，並且不需要創建補丁。例如，你可以通過在 `kustomization.yaml` 文件的 `images` 字段設置新的鏡像來 更改容器中使用的鏡像。

```bash
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
images:
- name: nginx
  newName: my.image.registry/nginx
  newTag: 1.4.0
EOF
```

執行 `kubectl kustomize ./` 以查看所使用的鏡像已被更新：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: my.image.registry/nginx:1.4.0
        name: my-nginx
        ports:
        - containerPort: 80
```

有些時候，Pod 中運行的應用可能需要使用來自其他對象的配置值。例如，某 Deployment 對象的 Pod 需要從環境變量或命令行參數中讀取讀取 Service 的名稱。由於在 `kustomization.yaml` 文件中添加 `namePrefix` 或 `nameSuffix` 時 Service 名稱可能發生變化，建議不要在命令參數中硬編碼 Service 名稱。對於這種使用場景，Kustomize 可以通過 `vars` 將 Service 名稱注入到容器中。

```bash
# 創建一個 deployment.yaml 文件（引用此處的文檔分隔符）
cat <<'EOF' > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        command: ["start", "--host", "$(MY_SERVICE_NAME)"]
EOF

# 創建一個 service.yaml 文件
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

cat <<EOF >./kustomization.yaml
namePrefix: dev-
nameSuffix: "-001"

resources:
- deployment.yaml
- service.yaml

vars:
- name: MY_SERVICE_NAME
  objref:
    kind: Service
    name: my-nginx
    apiVersion: v1
EOF
```

執行 `kubectl kustomize ./` 以查看注入到容器中的 Service 名稱是 `dev-my-nginx-001`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-my-nginx-001
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - command:
        - start
        - --host
        - dev-my-nginx-001
        image: nginx
        name: my-nginx
```

## 基準 (Bases) 與覆蓋 (Overlays)

Kustomize 中有 **基準（bases）** 和 **覆蓋（overlays）** 的概念區分。基準 是包含 `kustomization.yaml` 文件的一個目錄，其中包含一組資源及其相關的定制。基準可以是本地目錄或者來自遠程倉庫的目錄，只要其中存在 `kustomization.yaml` 文件即可。**覆蓋** 也是一個目錄，其中包含將其他 `kustomization` 目錄當做 `bases` 來引用的 `kustomization.yaml` 文件。基準不了解覆蓋的存在，且可被多個覆蓋所使用。覆蓋則可以有多個基準，且可針對所有基準中的資源執行組織操作，還可以在其上執行定制。

```bash
# 創建一個包含基準的目錄 
mkdir base
# 創建 base/deployment.yaml
cat <<EOF > base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
EOF

# 創建 base/service.yaml 文件
cat <<EOF > base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

# 創建 base/kustomization.yaml
cat <<EOF > base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

此基準可在多個覆蓋中使用。你可以在不同的覆蓋中添加不同的 namePrefix 或 其他貫穿性字段。下面是兩個使用同一基準的覆蓋：

```bash
mkdir dev
cat <<EOF > dev/kustomization.yaml
bases:
- ../base
namePrefix: dev-
EOF

mkdir prod
cat <<EOF > prod/kustomization.yaml
bases:
- ../base
namePrefix: prod-
EOF
```

## 如何使用 Kustomize 来应用、查看和删除对象

在 kubectl 命令中使用 `--kustomize` 或 `-k` 參數來識別被 `kustomization.yaml` 所管理的資源。注意 `-k` 要指向一個 `kustomization` 目錄。例如：

```bash
kubectl apply -k <kustomization 目錄>/
```

假定使用下面的 `kustomization.yaml`，

```bash
# 創建 deployment.yaml 文件
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# 創建 kustomization.yaml
cat <<EOF >./kustomization.yaml
namePrefix: dev-
commonLabels:
  app: my-nginx
resources:
- deployment.yaml
EOF
```

執行下面的命令來應用 Deployment 對象 dev-my-nginx：

```bash
kubectl apply -k ./
```

命令執行結果:

```
deployment.apps/dev-my-nginx created
```

運行下面的命令之一來查看 Deployment 對象 dev-my-nginx：

```bash
kubectl get -k ./
```

命令執行結果:

```
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
dev-my-nginx   2/2     2            2           75s
```

```bash
kubectl describe -k ./
```

執行下面的命令來比較 Deployment 對象 `dev-my-nginx` 與清單被應用之後 集群將處於的狀態：

```bash
kubectl diff -k ./
```

執行下面的命令刪除 Deployment 對象 dev-my-nginx：

```bash
kubectl delete -k ./
```

命令執行結果:

```
deployment.apps "dev-my-nginx" deleted
```

## Kustomize 功能特性列表

|字段	|類型	|解釋|
|---|---|---|
|namespace	|string	|為所有資源添加名字空間|
|namePrefix	|string	|此字段的值將被添加到所有資源名稱前面|
|nameSuffix	|string	|此字段的值將被添加到所有資源名稱後面|
|commonLabels	|map[string]string	|要添加到所有資源和選擇算符的標籤|
|commonAnnotations	|map[string]string	|要添加到所有資源的註解|
|resources	|[]string	|列表中的每個條目都必須能夠解析為現有的資源配置文件|
|configMapGenerator	|[]ConfigMapArgs	|列表中的每個條目都會生成一個 ConfigMap|
|secretGenerator	|[]SecretArgs	|列表中的每個條目都會生成一個 Secret|
|generatorOptions	|GeneratorOptions	|更改所有 ConfigMap 和 Secret 生成器的行為|
|bases	|[]string	|列表中每個條目都應能解析為一個包含 kustomization.yaml 文件的目錄|
|patchesStrategicMerge	|[]string	|列表中每個條目都能解析為某 Kubernetes 對象的策略性合併補丁|
|patchesJson6902	|[]Patch	|列表中每個條目都能解析為一個 Kubernetes 對象和一個 JSON 補丁|
|vars	|[]Var	|每個條目用來從某資源的字段來析取文字|
|images	|[]Image	|每個條目都用來更改鏡像的名稱、標記與/或摘要，不必生成補丁|
|configurations	|[]string	|列表中每個條目都應能解析為一個包含 Kustomize 轉換器配置 的文件|
|crds	|[]string	|列表中每個條目都應能夠解析為 Kubernetes 類別的 OpenAPI 定義文件|
