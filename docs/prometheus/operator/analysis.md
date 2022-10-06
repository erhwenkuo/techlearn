# Prometheus Operator Analysis

# Prometheus

執行下列命令來檢視 Prometheus:

```bash
kubectl get Prometheus  kube-stack-prometheus-kube-prometheus -o yaml
```

結果:

```yaml
kind: Prometheus
metadata:
  annotations:
    meta.helm.sh/release-name: kube-stack-prometheus
    meta.helm.sh/release-namespace: monitoring
  creationTimestamp: "2022-10-01T05:15:07Z"
  generation: 3
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: kube-stack-prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 40.1.2
    chart: kube-prometheus-stack-40.1.2
    heritage: Helm
    release: kube-stack-prometheus
  name: kube-stack-prometheus-kube-prometheus
  namespace: monitoring
  resourceVersion: "1666"
  uid: efec4085-21c2-46bf-9c9a-029c2caaac00
spec:
  alerting:
    alertmanagers:
    - apiVersion: v2
      name: kube-stack-prometheus-kube-alertmanager
      namespace: monitoring
      pathPrefix: /
      port: http-web
  enableAdminAPI: false
  evaluationInterval: 30s
  externalUrl: http://kube-stack-prometheus-kube-prometheus.monitoring:9090
  image: quay.io/prometheus/prometheus:v2.38.0
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  podMonitorNamespaceSelector: {}
  podMonitorSelector: {}
  portName: http-web
  probeNamespaceSelector: {}
  probeSelector: {}
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector: {}
  scrapeInterval: 30s
  securityContext:
    fsGroup: 2000
    runAsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: kube-stack-prometheus-kube-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  shards: 1
  version: v2.38.0
  walCompression: true
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2022-10-01T05:15:51Z"
    status: "True"
    type: Available
  - lastTransitionTime: "2022-10-01T05:15:21Z"
    status: "True"
    type: Reconciled
  paused: false
  replicas: 1
  shardStatuses:
  - availableReplicas: 1
    replicas: 1
    shardID: "0"
    unavailableReplicas: 0
    updatedReplicas: 1
  unavailableReplicas: 0
  updatedReplicas: 1
```

## Alertmanager

執行下列命令來檢視 Alertmanager:

```bash
kubectl get Alertmanager kube-stack-prometheus-kube-alertmanager -o yaml
```

結果:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  annotations:
    meta.helm.sh/release-name: kube-stack-prometheus
    meta.helm.sh/release-namespace: monitoring
  creationTimestamp: "2022-10-01T05:15:07Z"
  generation: 1
  labels:
    app: kube-prometheus-stack-alertmanager
    app.kubernetes.io/instance: kube-stack-prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 40.1.2
    chart: kube-prometheus-stack-40.1.2
    heritage: Helm
    release: kube-stack-prometheus
  name: kube-stack-prometheus-kube-alertmanager
  namespace: monitoring
  resourceVersion: "709"
  uid: 8092f00d-2077-445c-88d3-e533f2cd7fc2
spec:
  alertmanagerConfigNamespaceSelector: {}
  alertmanagerConfigSelector: {}
  # # Alertmanager 實例可用的外部 URL。這是生成正確 URL 所必需的。
  externalUrl: http://kube-stack-prometheus-kube-alertmanager.monitoring:9093
  image: quay.io/prometheus/alertmanager:v0.24.0
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  portName: http-web
  replicas: 1
  retention: 120h
  routePrefix: /
  securityContext:
    fsGroup: 2000
    runAsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: kube-stack-prometheus-kube-alertmanager
  version: v0.24.0
```

## ServiceMonitor

### kube-stack-prometheus-kube-kube-etcd

執行下列命令來檢視 ServiceMonitor:

```bash
kubectl get ServiceMonitor kube-stack-prometheus-kube-kube-etcd -o yaml
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  annotations:
    meta.helm.sh/release-name: kube-stack-prometheus
    meta.helm.sh/release-namespace: monitoring
  creationTimestamp: "2022-10-01T05:15:08Z"
  generation: 1
  labels:
    app: kube-prometheus-stack-kube-etcd
    app.kubernetes.io/instance: kube-stack-prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 40.1.2
    chart: kube-prometheus-stack-40.1.2
    heritage: Helm
    release: kube-stack-prometheus
  name: kube-stack-prometheus-kube-kube-etcd
  namespace: monitoring
  resourceVersion: "789"
  uid: 6526dc5b-ea9e-4ca1-a137-2a959e7c9bad
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    port: http-metrics
  jobLabel: jobLabel
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      app: kube-prometheus-stack-kube-etcd
      release: kube-stack-prometheus

```

### kube-stack-prometheus-kube-kubelet


執行下列命令來檢視 ServiceMonitor:

```bash
kubectl get ServiceMonitor kube-stack-prometheus-kube-kubelet -o yaml
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  annotations:
    meta.helm.sh/release-name: kube-stack-prometheus
    meta.helm.sh/release-namespace: monitoring
  creationTimestamp: "2022-10-01T11:21:22Z"
  generation: 1
  labels:
    app: kube-prometheus-stack-kubelet
    app.kubernetes.io/instance: kube-stack-prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 40.1.2
    chart: kube-prometheus-stack-40.1.2
    heritage: Helm
    release: kube-stack-prometheus
  name: kube-stack-prometheus-kube-kubelet
  namespace: monitoring
  resourceVersion: "625"
  uid: a2f83e06-39e8-45e3-b7af-8aae47928b68
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    honorLabels: true
    port: https-metrics
    relabelings:
    - action: replace
      sourceLabels:
      - __metrics_path__
      targetLabel: metrics_path
    scheme: https
    tlsConfig:
      caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      insecureSkipVerify: true
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    honorLabels: true
    metricRelabelings:
    - action: drop
      regex: container_cpu_(cfs_throttled_seconds_total|load_average_10s|system_seconds_total|user_seconds_total)
      sourceLabels:
      - __name__
    - action: drop
      regex: container_fs_(io_current|io_time_seconds_total|io_time_weighted_seconds_total|reads_merged_total|sector_reads_total|sector_writes_total|writes_merged_total)
      sourceLabels:
      - __name__
    - action: drop
      regex: container_memory_(mapped_file|swap)
      sourceLabels:
      - __name__
    - action: drop
      regex: container_(file_descriptors|tasks_state|threads_max)
      sourceLabels:
      - __name__
    - action: drop
      regex: container_spec.*
      sourceLabels:
      - __name__
    - action: drop
      regex: .+;
      sourceLabels:
      - id
      - pod
    path: /metrics/cadvisor
    port: https-metrics
    relabelings:
    - action: replace
      sourceLabels:
      - __metrics_path__
      targetLabel: metrics_path
    scheme: https
    tlsConfig:
      caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      insecureSkipVerify: true
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    honorLabels: true
    path: /metrics/probes
    port: https-metrics
    relabelings:
    - action: replace
      sourceLabels:
      - __metrics_path__
      targetLabel: metrics_path
    scheme: https
    tlsConfig:
      caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      insecureSkipVerify: true
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      app.kubernetes.io/name: kubelet
      k8s-app: kubelet
```

### kube-stack-prometheus-kube-prometheus


執行下列命令來檢視 ServiceMonitor:

```bash
kubectl get ServiceMonitor kube-stack-prometheus-kube-prometheus -o yaml
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  annotations:
    meta.helm.sh/release-name: kube-stack-prometheus
    meta.helm.sh/release-namespace: monitoring
  creationTimestamp: "2022-10-01T11:21:22Z"
  generation: 1
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: kube-stack-prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 40.1.2
    chart: kube-prometheus-stack-40.1.2
    heritage: Helm
    release: kube-stack-prometheus
  name: kube-stack-prometheus-kube-prometheus
  namespace: monitoring
  resourceVersion: "623"
  uid: fcd01228-daca-4176-8ae5-755e21c91696
spec:
  endpoints:
  - path: /metrics
    port: http-web
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      app: kube-prometheus-stack-prometheus
      release: kube-stack-prometheus
      self-monitor: "true"
```

## PrometheusRule

### kube-stack-prometheus-kube-prometheus

Prometheus 的告警規則:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  annotations:
    meta.helm.sh/release-name: kube-stack-prometheus
    meta.helm.sh/release-namespace: monitoring
  creationTimestamp: "2022-10-01T11:21:22Z"
  generation: 1
  labels:
    app: kube-prometheus-stack
    app.kubernetes.io/instance: kube-stack-prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 40.1.2
    chart: kube-prometheus-stack-40.1.2
    heritage: Helm
    release: kube-stack-prometheus
  name: kube-stack-prometheus-kube-prometheus
  namespace: monitoring
  resourceVersion: "617"
  uid: d3b10750-992e-41e4-a9ae-6c40f2a9a476
spec:
  groups:
  - name: prometheus
    rules:
    - alert: PrometheusBadConfig
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has failed to
          reload its configuration.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusbadconfig
        summary: Failed Prometheus configuration reload.
      expr: |-
        # Without max_over_time, failed scrapes could create false negatives, see
        # https://www.robustperception.io/alerting-on-gauges-in-prometheus-2-0 for details.
        max_over_time(prometheus_config_last_reload_successful{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m]) == 0
      for: 10m
      labels:
        severity: critical
    - alert: PrometheusNotificationQueueRunningFull
      annotations:
        description: Alert notification queue of Prometheus {{$labels.namespace}}/{{$labels.pod}}
          is running full.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusnotificationqueuerunningfull
        summary: Prometheus alert notification queue predicted to run full in less
          than 30m.
      expr: |-
        # Without min_over_time, failed scrapes could create false negatives, see
        # https://www.robustperception.io/alerting-on-gauges-in-prometheus-2-0 for details.
        (
          predict_linear(prometheus_notifications_queue_length{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m], 60 * 30)
        >
          min_over_time(prometheus_notifications_queue_capacity{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        )
      for: 15m
      labels:
        severity: warning
    - alert: PrometheusErrorSendingAlertsToSomeAlertmanagers
      annotations:
        description: '{{ printf "%.1f" $value }}% errors while sending alerts from
          Prometheus {{$labels.namespace}}/{{$labels.pod}} to Alertmanager {{$labels.alertmanager}}.'
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheuserrorsendingalertstosomealertmanagers
        summary: Prometheus has encountered more than 1% errors sending alerts to
          a specific Alertmanager.
      expr: |-
        (
          rate(prometheus_notifications_errors_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        /
          rate(prometheus_notifications_sent_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        )
        * 100
        > 1
      for: 15m
      labels:
        severity: warning
    - alert: PrometheusNotConnectedToAlertmanagers
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} is not connected
          to any Alertmanagers.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusnotconnectedtoalertmanagers
        summary: Prometheus is not connected to any Alertmanagers.
      expr: |-
        # Without max_over_time, failed scrapes could create false negatives, see
        # https://www.robustperception.io/alerting-on-gauges-in-prometheus-2-0 for details.
        max_over_time(prometheus_notifications_alertmanagers_discovered{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m]) < 1
      for: 10m
      labels:
        severity: warning
    - alert: PrometheusTSDBReloadsFailing
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has detected
          {{$value | humanize}} reload failures over the last 3h.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheustsdbreloadsfailing
        summary: Prometheus has issues reloading blocks from disk.
      expr: increase(prometheus_tsdb_reloads_failures_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[3h])
        > 0
      for: 4h
      labels:
        severity: warning
    - alert: PrometheusTSDBCompactionsFailing
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has detected
          {{$value | humanize}} compaction failures over the last 3h.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheustsdbcompactionsfailing
        summary: Prometheus has issues compacting blocks.
      expr: increase(prometheus_tsdb_compactions_failed_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[3h])
        > 0
      for: 4h
      labels:
        severity: warning
    - alert: PrometheusNotIngestingSamples
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} is not ingesting
          samples.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusnotingestingsamples
        summary: Prometheus is not ingesting samples.
      expr: |-
        (
          rate(prometheus_tsdb_head_samples_appended_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m]) <= 0
        and
          (
            sum without(scrape_job) (prometheus_target_metadata_cache_entries{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}) > 0
          or
            sum without(rule_group) (prometheus_rule_group_rules{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}) > 0
          )
        )
      for: 10m
      labels:
        severity: warning
    - alert: PrometheusDuplicateTimestamps
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} is dropping
          {{ printf "%.4g" $value  }} samples/s with different values but duplicated
          timestamp.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusduplicatetimestamps
        summary: Prometheus is dropping samples with duplicate timestamps.
      expr: rate(prometheus_target_scrapes_sample_duplicate_timestamp_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        > 0
      for: 10m
      labels:
        severity: warning
    - alert: PrometheusOutOfOrderTimestamps
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} is dropping
          {{ printf "%.4g" $value  }} samples/s with timestamps arriving out of order.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusoutofordertimestamps
        summary: Prometheus drops samples with out-of-order timestamps.
      expr: rate(prometheus_target_scrapes_sample_out_of_order_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        > 0
      for: 10m
      labels:
        severity: warning
    - alert: PrometheusRemoteStorageFailures
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} failed to send
          {{ printf "%.1f" $value }}% of the samples to {{ $labels.remote_name}}:{{
          $labels.url }}
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusremotestoragefailures
        summary: Prometheus fails to send samples to remote storage.
      expr: |-
        (
          (rate(prometheus_remote_storage_failed_samples_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m]) or rate(prometheus_remote_storage_samples_failed_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m]))
        /
          (
            (rate(prometheus_remote_storage_failed_samples_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m]) or rate(prometheus_remote_storage_samples_failed_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m]))
          +
            (rate(prometheus_remote_storage_succeeded_samples_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m]) or rate(prometheus_remote_storage_samples_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m]))
          )
        )
        * 100
        > 1
      for: 15m
      labels:
        severity: critical
    - alert: PrometheusRemoteWriteBehind
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} remote write
          is {{ printf "%.1f" $value }}s behind for {{ $labels.remote_name}}:{{ $labels.url
          }}.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusremotewritebehind
        summary: Prometheus remote write is behind.
      expr: |-
        # Without max_over_time, failed scrapes could create false negatives, see
        # https://www.robustperception.io/alerting-on-gauges-in-prometheus-2-0 for details.
        (
          max_over_time(prometheus_remote_storage_highest_timestamp_in_seconds{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        - ignoring(remote_name, url) group_right
          max_over_time(prometheus_remote_storage_queue_highest_sent_timestamp_seconds{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        )
        > 120
      for: 15m
      labels:
        severity: critical
    - alert: PrometheusRemoteWriteDesiredShards
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} remote write
          desired shards calculation wants to run {{ $value }} shards for queue {{
          $labels.remote_name}}:{{ $labels.url }}, which is more than the max of {{
          printf `prometheus_remote_storage_shards_max{instance="%s",job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}`
          $labels.instance | query | first | value }}.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusremotewritedesiredshards
        summary: Prometheus remote write desired shards calculation wants to run more
          than configured max shards.
      expr: |-
        # Without max_over_time, failed scrapes could create false negatives, see
        # https://www.robustperception.io/alerting-on-gauges-in-prometheus-2-0 for details.
        (
          max_over_time(prometheus_remote_storage_shards_desired{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        >
          max_over_time(prometheus_remote_storage_shards_max{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        )
      for: 15m
      labels:
        severity: warning
    - alert: PrometheusRuleFailures
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has failed to
          evaluate {{ printf "%.0f" $value }} rules in the last 5m.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusrulefailures
        summary: Prometheus is failing rule evaluations.
      expr: increase(prometheus_rule_evaluation_failures_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        > 0
      for: 15m
      labels:
        severity: critical
    - alert: PrometheusMissingRuleEvaluations
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has missed {{
          printf "%.0f" $value }} rule group evaluations in the last 5m.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusmissingruleevaluations
        summary: Prometheus is missing rule evaluations due to slow rule group evaluation.
      expr: increase(prometheus_rule_group_iterations_missed_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        > 0
      for: 15m
      labels:
        severity: warning
    - alert: PrometheusTargetLimitHit
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has dropped
          {{ printf "%.0f" $value }} targets because the number of targets exceeded
          the configured target_limit.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheustargetlimithit
        summary: Prometheus has dropped targets because some scrape configs have exceeded
          the targets limit.
      expr: increase(prometheus_target_scrape_pool_exceeded_target_limit_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        > 0
      for: 15m
      labels:
        severity: warning
    - alert: PrometheusLabelLimitHit
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has dropped
          {{ printf "%.0f" $value }} targets because some samples exceeded the configured
          label_limit, label_name_length_limit or label_value_length_limit.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheuslabellimithit
        summary: Prometheus has dropped targets because some scrape configs have exceeded
          the labels limit.
      expr: increase(prometheus_target_scrape_pool_exceeded_label_limits_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        > 0
      for: 15m
      labels:
        severity: warning
    - alert: PrometheusScrapeBodySizeLimitHit
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has failed {{
          printf "%.0f" $value }} scrapes in the last 5m because some targets exceeded
          the configured body_size_limit.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusscrapebodysizelimithit
        summary: Prometheus has dropped some targets that exceeded body size limit.
      expr: increase(prometheus_target_scrapes_exceeded_body_size_limit_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        > 0
      for: 15m
      labels:
        severity: warning
    - alert: PrometheusScrapeSampleLimitHit
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has failed {{
          printf "%.0f" $value }} scrapes in the last 5m because some targets exceeded
          the configured sample_limit.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusscrapesamplelimithit
        summary: Prometheus has failed scrapes that have exceeded the configured sample
          limit.
      expr: increase(prometheus_target_scrapes_exceeded_sample_limit_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        > 0
      for: 15m
      labels:
        severity: warning
    - alert: PrometheusTargetSyncFailure
      annotations:
        description: '{{ printf "%.0f" $value }} targets in Prometheus {{$labels.namespace}}/{{$labels.pod}}
          have failed to sync because invalid configuration was supplied.'
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheustargetsyncfailure
        summary: Prometheus has failed to sync targets.
      expr: increase(prometheus_target_sync_failed_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[30m])
        > 0
      for: 5m
      labels:
        severity: critical
    - alert: PrometheusHighQueryLoad
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} query API has
          less than 20% available capacity in its query engine for the last 15 minutes.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheushighqueryload
        summary: Prometheus is reaching its maximum capacity serving concurrent requests.
      expr: avg_over_time(prometheus_engine_queries{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        / max_over_time(prometheus_engine_queries_concurrent_max{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring"}[5m])
        > 0.8
      for: 15m
      labels:
        severity: warning
    - alert: PrometheusErrorSendingAlertsToAnyAlertmanager
      annotations:
        description: '{{ printf "%.1f" $value }}% minimum errors while sending alerts
          from Prometheus {{$labels.namespace}}/{{$labels.pod}} to any Alertmanager.'
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheuserrorsendingalertstoanyalertmanager
        summary: Prometheus encounters more than 3% errors sending alerts to any Alertmanager.
      expr: |-
        min without (alertmanager) (
          rate(prometheus_notifications_errors_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring",alertmanager!~``}[5m])
        /
          rate(prometheus_notifications_sent_total{job="kube-stack-prometheus-kube-prometheus",namespace="monitoring",alertmanager!~``}[5m])
        )
        * 100
        > 3
      for: 15m
      labels:
        severity: critical
```

defaultRules.runbookUrl