# 使用 Prometheus 和 Grafana 監控 PostgreSQL

![](./assets/observability-metrics.png)

在使用 PostgreSQL 過程中都需要對 PostgreSQL 運行狀態進行監控，以便了解 PostgreSQL 服務是否運行正常，排查 PostgreSQL 故障問題原因， Prometheus 監控服務提供了基於 Exporter 的方式來監控 PostgreSQL 運行狀態，並提供了開箱即用的 Grafana 監控儀表板。本文介紹如何部署 Exporter 以及實現 PostgreSQL Exporter 告警接入等操作。

- 參考:[postgres-exporter-quickstart](https://grafana.com/oss/prometheus/exporters/postgres-exporter/#postgres-exporter-quickstart)

## 步驟 01 - 環境安裝

### Kubernetes

本教程使用 K3D 來構建實驗 K8S 集群, 詳細說明請參考: 

- [使用 K3D 設置 Kubernetes 集群](../../kubernetes/01-getting-started/learning-env/k3d/k3s-kubernetes-cluster-setup-with-k3d.md)

執行下列命令來創建實驗 Kubernetes 集群:

```bash
k3d cluster create --api-port 6443 \
--port 8080:80@loadbalancer --port 8443:443@loadbalancer
```

確認 Kubernetes 及 Kubectl 是否成功安裝：

```bash
kubectl cluster-info
```
(輸出結果)

```bash
Kubernetes control plane is running at https://0.0.0.0:6443
CoreDNS is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

### kube-prometheus-stack

本教程使用 `kube-prometheus-stack` 來構建可觀測性的相關元件, 詳細說明請參考:

- [Prometheus 簡介](../prometheus/overview.md)
- [Prometheus Operator 簡介](../operator/install.md)

添加 Prometheus-Community helm 存儲庫並更新本地緩存：

```bash
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update 
```

創建要配置的 vlaues 檔案:

```yaml title="kube-stack-prometheus-values.yaml"
grafana:
  # change timezone setting base on browser
  defaultDashboardsTimezone: browser
  grafana.ini:
    users:
      viewers_can_edit: true
    auth:
      disable_login_form: false
      disable_signout_menu: false
    auth.anonymous:
      enabled: true
      org_role: Viewer
  sidecar:
    datasources:
      logLevel: "DEBUG"
      enabled: true
      searchNamespace: "ALL"
    dashboards:
      logLevel: "DEBUG"
      # enable the cluster wide search for dashbaords and adds/updates/deletes them in grafana
      enabled: true
      searchNamespace: "ALL"
      label: grafana_dashboard
      labelValue: "1"

prometheus:
  prometheusSpec:
    # enable the cluster wide search for ServiceMonitor CRD
    serviceMonitorSelectorNilUsesHelmValues: false
    # enable the cluster wide search for PodMonitor CRD
    podMonitorSelectorNilUsesHelmValues: false
    # enable the cluster wide search for PrometheusRule CRD
    ruleSelectorNilUsesHelmValues: false
    probeSelectorNilUsesHelmValues: false
```

使用 Helm 在命名空間監控中部署 `kube-stack-prometheus` chart：

```bash
helm upgrade --install --wait --create-namespace --namespace monitoring  \
kube-stack-prometheus prometheus-community/kube-prometheus-stack \
--values kube-stack-prometheus-values.yaml
```

該 Helm chart 安裝了 Prometheus 組件和 Operator、Grafana 以及以下 exporters：

- [prometheus-node-exporter](https://github.com/prometheus/node_exporter) 暴露底層硬件和操作系統的相關指標
- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) 監聽 Kubernetes API 服務器並生成有關對象狀態的指標

有關 `kube-stack-prometheus` 的詳細說明:

- [Prometheus Operator](../operator/install.md)

#### 連接到 Prometheus

Prometheus Web UI 可通過以下命令通過端口轉發訪問：

```bash
kubectl port-forward --namespace monitoring \
svc/kube-stack-prometheus-kube-prometheus 9090:9090 --address="0.0.0.0"
```

在 http://localhost:9090 上打開瀏覽器選項卡會顯示 Prometheus Web UI。我們可以檢索從不同指標 Exporters 所收集回來的指標：

![](./assets/prometheus-ui.png)

#### 連接到 AlertManager

AlertManager Web UI 可通過以下命令通過端口轉發訪問：

```bash
kubectl port-forward --namespace monitoring \
svc/kube-stack-prometheus-kube-alertmanager 9093:9093 --address="0.0.0.0"
```

![](./assets/alertmanager-ui.png)

#### 連接到 Grafana

Grafana Web UI 可通過以下命令通過端口轉發訪問：

```bash
kubectl port-forward --namespace monitoring \
svc/kube-stack-prometheus-grafana 3000:80 --address="0.0.0.0"
```

打開瀏覽器並轉到 http://localhost:3000 並填寫前一個命令所取得的用戶名/密碼。預設的帳號是:

- `username`: admin
- `password`: prom-operator


![](./assets/grafana-ui.png)

## 步驟 02 - 監控 PostgreSQL 實戰

!!! info
    在本教程裡要監控的 PostgreSQL 的數據庫是佈署在 Kubernetes 集群外面。

![](./assets/observability-metrics-details.png)

整合的手法可簡化成五個步驟:

1. 安裝設定 Postgres Exporter
2. 配置 Service 與 Endpoint
3. 設定 Postgres 指標的收集
4. 設定 Postgres 告警規則
5. 創建 Postgres 監控儀表板

### 1. 安裝設定 Postgres Exporter

本教程如何使用 docker-compose 來構建一個 Postgres DB的實例並且配置了 postgres-export: 

```yaml title="docker-compose.yaml" hl_lines="19 20"
version: "3.7"
services:
  postgresdb:
    image: postgres:13.2-alpine
    restart: always
    environment:
        POSTGRES_DB: postgres
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: password123
        PGDATA: /var/lib/postgresql/data
    volumes:
        - db-data:/var/lib/postgresql/data
    ports:
        - "5432:5432"
  postgres-exporter:
    image: quay.io/prometheuscommunity/postgres-exporter
    restart: always
    environment:
        DATA_SOURCE_NAME: postgresql://postgres:dxlab@postgresdb:5432/postgres?sslmode=disable
        PG_EXPORTER_EXTEND_QUERY_PATH: /extend-query/queries.yaml
    volumes:
      - ./queries.yaml:/extend-query/queries.yaml
    ports:
        - "9187:9187"
volumes:
    db-data:
```

由於 postgres-export 要通過一個 `queries.yaml` 的結構來定義相關的指標。下面列舉三個可下載 `queries.yaml` 的參考:

- Postgres Export 官網範例: [queries.yaml](https://github.com/prometheus-community/postgres_exporter/blob/master/queries.yaml)
- Gitlab 範例: [Additions to queries.yml for the postgres exporter](https://gitlab.com/gitlab-org/omnibus-gitlab/-/issues/6345)
- 本教程範例: [queries.yaml](./assets/queries.yaml)

下載 `queries.yaml` 至 `docker-compose.yaml` 的同一個目錄下，接著執行下列指令來啟動 Postgres 與 PostgresExporter:

```bash
$ docker-compose up -d

Creating network "ws_pg_default" with the default driver
Creating volume "ws_pg_db-data" with default driver
Creating ws_pg_postgresdb_1        ... done
Creating ws_pg_postgres-exporter_1 ... done
```

使用瀏覽器打開 `http://localhost:9187/metric` 便可看到相關由 PostgresExporter 曝露出來的指標:

![](./assets/metrics-postgres-exporter.png)

### 2. 配置 Service 與 Endpoint

許多佈署在 Kubernetes 裡的應用程序會使用到外部資源，例如外部數據庫或外部 SaaS 服務。這些外部資源可以配置成為 Kubernetes 裡的 `Service`，以便應用程序可以像使用任何其他 Kubernetes 內部服務一樣使用它們。

**為外部 Postgres 數據庫定義服務**

您可以通過提供 IP 地址和端點，或通過提供完全限定的域名 (FQDN) 來定義服務。本教程示範使用 IP 地址和端點來進行配置:

```bash title="Service"
kubectl apply -f -<<EOF
kind: Service
apiVersion: v1
metadata:
  name: external-pg-exporter
  labels:
    database: testdb
spec:
  selector: {}
  ports:
    -
      name: pg
      protocol: TCP
      port: 9187
EOF
```

!!! tip
    由於外部資源的服務並不需要關聯到 pod。因此對這個 Service 不要設置選擇器 `selector: {}`。

接下來，為服務創建所需的端點。這為服務代理和路由器提供了向服務發送流量的位置：

```bash title="Endpoints"
kubectl apply -f -<<EOF
kind: Endpoints
apiVersion: v1
metadata:
  name: external-pg-exporter
subsets: 
  - addresses:
      # setup the real ip address for the extrnal postgre database
      - ip: 192.168.53.208
    ports:
      - name: pg
        port: 9187
EOF
```

注意事項:

- 服務實例的名稱，如上一步中所定義。
- 如果提供多個端點，服務的流量將在提供的端點之間進行負載平衡。
- 端點 IP 不能是 loopback (127.0.0.0/8)、link-local (169.254.0.0/16) 或 link-local multicast (224.0.0.0/24)。
- 端口和名稱定義必須與上一步中定義的服務中的端口和名稱值相匹配。

### 3. 設定 Postgres 指標的收集

在本教程中 Prometheus 是用來收集應用程式指標數據的元件，而 Prometheus 是使用拉 (pull) 的手法來取得各個應用的指標。在 `kube-prometheus-stack` 的安裝包裡頭也安裝了 `prometheus-operator` 的元件與相關的 CRD 來讓各應用程式的開發團隊可使用這些 CRD　來自服務式地配置讓 Prometheus 可認知到如何 scrape 到應用程序的指標 endpoint。

詳細的說明請參考: 

- [自定義監控報警](../operator/custom.md)

接下來讓我們應用 `ServiceMonitor` (CRD) 來讓 Prometheus Operator 來幫我們設定應用程序指標數據的收集。

構建一個 ServiceMonitor 對象來配置 Prometheus 來刮取應用程序的指標數據:

```bash title="ServiceMonitor"
kubectl apply -f -<<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: external-pg-exporter
  labels:
    database: testdb
spec:
  selector:
    matchLabels:
      database: testdb
  namespaceSelector:
    matchNames:
    - default
  endpoints:
  - port: pg
    path: /metrics
EOF
```

驗證:

在 Prometheus UI　中點選 `Status >> Target`:

![](./assets/servicemonitor-pg-exporter.png)

### 4. 設定 Postgres 告警規則

Prometheus 除了可以收集應用程式指標,同時它也是用來執行告警規則的元件。`prometheus-operator` 的 CRD 中 `PrometheusRule` 就是讓各應用程式的開發團隊可以使用這個 CRD 來自服務式地配置讓 Prometheus 執行的告警規則。

詳細的說明請參考: 

- [自定義監控報警](../operator/custom.md)

接下來讓我們應用 `PrometheusRule` (CRD) 來讓 Prometheus Operator 來幫我們設定告警規則。

Prometheus Alert Rules 主要參考了:

- [Awesome Prometheus alerts](https://awesome-prometheus-alerts.grep.to/rules#postgresql)
- [Grafana Lab - Postgres Exporter](https://grafana.com/oss/prometheus/exporters/postgres-exporter/?tab=alerting-rules#postgres-exporter-quickstart)

```yaml title="pg_alertrules.yaml"
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    team: dxlab
  name: pg-alertrules
spec:
  groups:
  - name: PostgresExporter
    rules:
      - alert: PostgresqlDown
        expr: 'pg_up == 0'
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Postgresql down (instance {{ $labels.instance }})
          description: "Postgresql instance is down\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlRestarted
        expr: 'time() - pg_postmaster_start_time_seconds < 60'
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Postgresql restarted (instance {{ $labels.instance }})
          description: "Postgresql restarted\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlExporterError
        expr: 'pg_exporter_last_scrape_error > 0'
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Postgresql exporter error (instance {{ $labels.instance }})
          description: "Postgresql exporter is showing errors. A query may be buggy in query.yaml\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlTableNotAutoVacuumed
        expr: '(pg_stat_user_tables_last_autovacuum > 0) and (time() - pg_stat_user_tables_last_autovacuum) > 60 * 60 * 24 * 10'
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Postgresql table not auto vacuumed (instance {{ $labels.instance }})
          description: "Table {{ $labels.relname }} has not been auto vacuumed for 10 days\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlTableNotAutoAnalyzed
        expr: '(pg_stat_user_tables_last_autoanalyze > 0) and (time() - pg_stat_user_tables_last_autoanalyze) > 24 * 60 * 60 * 10'
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Postgresql table not auto analyzed (instance {{ $labels.instance }})
          description: "Table {{ $labels.relname }} has not been auto analyzed for 10 days\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlTooManyConnections
        expr: 'sum by (datname) (pg_stat_activity_count{datname!~"template.*|postgres"}) > pg_settings_max_connections * 0.8'
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Postgresql too many connections (instance {{ $labels.instance }})
          description: "PostgreSQL instance has too many connections (> 80%).\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlNotEnoughConnections
        expr: 'sum by (datname) (pg_stat_activity_count{datname!~"template.*|postgres"}) < 5'
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Postgresql not enough connections (instance {{ $labels.instance }})
          description: "PostgreSQL instance should have more connections (> 5)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlDeadLocks
        expr: 'increase(pg_stat_database_deadlocks{datname!~"template.*|postgres"}[1m]) > 5'
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Postgresql dead locks (instance {{ $labels.instance }})
          description: "PostgreSQL has dead-locks\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlHighRollbackRate
        expr: 'rate(pg_stat_database_xact_rollback{datname!~"template.*"}[3m]) / rate(pg_stat_database_xact_commit{datname!~"template.*"}[3m]) > 0.02'
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Postgresql high rollback rate (instance {{ $labels.instance }})
          description: "Ratio of transactions being aborted compared to committed is > 2 %\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlCommitRateLow
        expr: 'rate(pg_stat_database_xact_commit[1m]) < 10'
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: Postgresql commit rate low (instance {{ $labels.instance }})
          description: "Postgresql seems to be processing very few transactions\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlLowXidConsumption
        expr: 'rate(pg_txid_current[1m]) < 5'
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Postgresql low XID consumption (instance {{ $labels.instance }})
          description: "Postgresql seems to be consuming transaction IDs very slowly\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlHighRateStatementTimeout
        expr: 'rate(postgresql_errors_total{type="statement_timeout"}[1m]) > 3'
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Postgresql high rate statement timeout (instance {{ $labels.instance }})
          description: "Postgres transactions showing high rate of statement timeouts\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlHighRateDeadlock
        expr: 'increase(postgresql_errors_total{type="deadlock_detected"}[1m]) > 1'
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Postgresql high rate deadlock (instance {{ $labels.instance }})
          description: "Postgres detected deadlocks\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlUnusedReplicationSlot
        expr: 'pg_replication_slots_active == 0'
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: Postgresql unused replication slot (instance {{ $labels.instance }})
          description: "Unused Replication Slots\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlTooManyDeadTuples
        expr: '((pg_stat_user_tables_n_dead_tup > 10000) / (pg_stat_user_tables_n_live_tup + pg_stat_user_tables_n_dead_tup)) >= 0.1'
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Postgresql too many dead tuples (instance {{ $labels.instance }})
          description: "PostgreSQL dead tuples is too large\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlConfigurationChanged
        expr: '{__name__=~"pg_settings_.*"} != ON(__name__) {__name__=~"pg_settings_([^t]|t[^r]|tr[^a]|tra[^n]|tran[^s]|trans[^a]|transa[^c]|transac[^t]|transact[^i]|transacti[^o]|transactio[^n]|transaction[^_]|transaction_[^r]|transaction_r[^e]|transaction_re[^a]|transaction_rea[^d]|transaction_read[^_]|transaction_read_[^o]|transaction_read_o[^n]|transaction_read_on[^l]|transaction_read_onl[^y]).*"} OFFSET 5m'
        for: 0m
        labels:
          severity: info
        annotations:
          summary: Postgresql configuration changed (instance {{ $labels.instance }})
          description: "Postgres Database configuration change has occurred\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlSslCompressionActive
        expr: 'sum(pg_stat_ssl_compression) > 0'
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Postgresql SSL compression active (instance {{ $labels.instance }})
          description: "Database connections with SSL compression enabled. This may add significant jitter in replication delay. Replicas should turn off SSL compression via `sslcompression=0` in `recovery.conf`.\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlTooManyLocksAcquired
        expr: '((sum (pg_locks_count)) / (pg_settings_max_locks_per_transaction * pg_settings_max_connections)) > 0.20'
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: Postgresql too many locks acquired (instance {{ $labels.instance }})
          description: "Too many locks acquired on the database. If this alert happens frequently, we may need to increase the postgres setting max_locks_per_transaction.\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlBloatIndexHigh(>80%)
        expr: 'pg_bloat_btree_bloat_pct > 80 and on (idxname) (pg_bloat_btree_real_size > 100000000)'
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: Postgresql bloat index high (> 80%) (instance {{ $labels.instance }})
          description: "The index {{ $labels.idxname }} is bloated. You should execute `REINDEX INDEX CONCURRENTLY {{ $labels.idxname }};`\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

      - alert: PostgresqlBloatTableHigh(>80%)
        expr: 'pg_bloat_table_bloat_pct > 80 and on (relname) (pg_bloat_table_real_size > 200000000)'
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: Postgresql bloat table high (> 80%) (instance {{ $labels.instance }})
          description: "The table {{ $labels.relname }} is bloated. You should execute `VACUUM {{ $labels.relname }};`\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```

執行下列命令來創建相關的告警規則:

```bash
$ kubectl apply -f pg_alertrules.yaml

prometheusrule.monitoring.coreos.com/pg-alertrules created
```

驗證:

在 Prometheus UI　中點選 `Status >> Rules`:

![](./assets/prometheusrule-postgres-exporter.png)

### 5. 創建 Postgres 監控儀表板

Grafana 確實是一個了不起的、強大的工具，它可以通過漂亮、靈活的儀表板創建、探索和共享您的數據——無論數據存儲在哪裡。

但是，隨著公司的發展，在維護大量儀表板方面帶來了新的挑戰。讓我們來看看其中的一些……

1. `Inconsistent dashboards in different environments`: 很多時候，我們根據當時的特定需求更新儀表板，然後忘記在所有其他環境中更新它。這會導致跨環境的不一致，因此它使監控變得更加困難，有時甚至效率低下。

2. `Ownership`: 儀表板沒有明確的所有者。創建和更改儀表板，與佈署儀表板的權責不明。

3. `Diaster recovery`: 當涉及到災難恢復、創建新環境或將現有環境遷移到新集群時，如何將所有內容恢復到所需狀態並準備必要的調整。

4. `ChangeLog`: 隨著開發部門的發展，儀表板的變化也越來越頻繁。有時，可以針對特定案例或測試在不同環境中更改多個儀表板。 Grafana 有自己的變更管理，但是，它是每個儀表板的。我們缺乏一個集中的地方，在那裡我們可以看到某人所做的所有更改，並且能夠在需要時立即回滾所有內容。

#### Dashboards, ConfigMaps 與 GitOps

幸運的是，Grafana 5.0 版引入了一個使用配置文件的新的主動供應系統。這意味著我們現在能夠從配置文件（Kubernetes 中的 ConfigMap）配置儀表板。 Grafana 將自動加載它們，而無需進一步部署部署（就像我們過去在 Kubernetes 中使用配置映射時一樣）。

我們現在需要找到一種方法將 configmap 從一個地方分發到所有環境。GitOps 方法可以幫助我們實現這一點，因為我們將儀表板配置存儲在 git 中，合併到主分支的任何更改都將自動應用於我們的系統。

市面上有很多 GitOps 工具，比如 Flux、Jenkins 等等——我們選擇了 ArgoCD。ArgoCD 是用於 Kubernetes 的聲明式 GitOps 持續交付工具。

ArgoCD 持續監視指定的 git 存儲庫，將其與集群中當前部署的清單進行比較，並在它們上有效地運行 `kubectl apply -f <all>`。

對於廣泛的用例，這導致了一個非常簡單和可見的持續交付過程——在我們的例子中，在 configmaps 中持續部署 Grafana 儀表板。

![](./assets/grafana-dashboard-gitop.png)


想了解如何將自己客制的 Dashbaord 轉換成 Configmap 來 GitOps 進到 Kubernetes 中的 Grafana，請參考: [使用 ConfigMap 添加 Grafana 儀表板](../operator/dashboard-using-configmap.md)。

接下來我們會使用 [Grafana Lab - Dashbaord](https://grafana.com/grafana/dashboards/) 網站上公開的 Dashbaord 來作示範。


**搜索儀表板:**

在 `filter` 上鍵入 `postgres`:

![](./assets/postgres-dashboard-search.png)

點選合適的儀表板(例如有 936K download的 [PostgreSQL Database](https://grafana.com/grafana/dashboards/9628-postgresql-database/)):

![](./assets/postgres-dashboard-search2.png)

點選 `Copy ID to clipboard`。

**創建儀表板:**

登入至一個 **開發用** 的 Grafana (開發者需要被授與 [Grafana `Editor` 權限](https://grafana.com/docs/grafana/latest/administration/roles-and-permissions/)),　然後在 Grafana UI　中的左側瀏覽欄中點選 "Dashboards >> Import", 並在 `Import via grafana.com` 欄中輸入上一步驟所複製下來的 Dashboard ID, 然後點選 "Load":

![](./assets/postgres-grafana-load-dashboard.png)

接著點選 `Select a Prometheus data source` 的下拉選單來選擇在本地 Grafana 配置好的 Prometheus data soucre 之後再點選 `Import`:

![](./assets/postgres-dashboard-import2.png)

**導出儀表板:**

在儀表板上方點選 `Share dashboard or panel` 的圖示:

![](./assets/postgres-dashboard-share.png)

切換到 `Export` 頁籤之後再點選 `Save to file` 來下載儀表板:

![](./assets/postgres-dashboard-to-file.png)

**構建儀表板 ConfigMap:**

由於下載的檔名裡包含許多空白字元，修改檔名便於後續的操作：

```bash
mv 'PostgreSQL Database-1670570791022.json' pg-dashboard.json
```

執行下列指令來創建 configmap 的 yaml 檔案:

```bash
kubectl create configmap cm-pg-dashboard \
--from-file=pg-dashboard.json \
--dry-run=client -o yaml > cm-pg-dashboard.yaml
```

結果會在當前的目錄下產生出一個包含 Dashboard 的 ConfigMap 檔案 `cm-pg-dashboard.yaml`。

使用任一文件編輯器來編修 `cm-pg-dashboard.yaml` 來增加一個關鍵的標籤`{==grafana_dashboard: "1"==}`:

```yaml title="cm-pg-dashboard.yaml" hl_lines="13 14"
apiVersion: v1
data:
  pg-dashboard.json: |-
    {
    ...
    ...
    ...
    }
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: cm-pg-dashboard
  labels:
    grafana_dashboard: "1"
```

下載範例Dashboard: [cm-pg-dashboard.yaml]()

執行下列指令來手動將 `cm-pg-dashboard.yaml` 建置進 Kubernetes 中:

```bash
kubectl apply -f cm-pg-dashboard.yaml
```

**在 Grafana 中驗證儀表板:**

在 Grafana UI　中的左側瀏覽欄中點選 "Dashboards >> Browse", 並在 filter 欄中輸入 "postgres":

![](./assets/postgres-dashboard-imported2.png)

**在 Grafana 中驗證儀表板:**

打開儀表板之後點選上方　"Save dashoard"　的圖示, 接著 Grafa 會從右側滑動一個視窗來說明這是一個 `Provisioned dashboard`:

![](./assets/postgres-dashboard-provisioned.png)

## 結論

在本教程，您在本機上使用 Docker-Compose 安裝並運行了 Postgres Exporter。然後，使用 ServiceMonitor 配置 Prometheus 為抓取 Postgres Exporter 公開的數據庫和 Postgres 指標。您將警報規則加載到 Prometheus 中，最後導入了 Grafana 儀表板以可視化您的 Postgres 指標。