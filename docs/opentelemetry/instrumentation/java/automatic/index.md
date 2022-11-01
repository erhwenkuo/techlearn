# Automatic Instrumentation
 
Java 自動檢測使用可附加到任何 Java 8+ 應用程序的 Java 代理 JAR。它動態注入 bytecode 以從許多流行的庫和框架中捕獲遙測數據。它可用於在應用程序或服務的“邊緣”捕獲遙測數據，例如入站請求、出站 HTTP 調用、數據庫調用等。要了解如何手動檢測您的服務或應用程序代碼，請參閱[手動檢測](https://opentelemetry.io/docs/instrumentation/java/manual)。

## 設置

1. 從 `opentelemetry-java-instrumentation` 存儲庫的 Releases 下載 [opentelemetry-javaagent.jar](https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar)。 JAR 文件包含代理和檢測庫。

2. 將 JAR 放在您的目錄中並使用您的應用程序啟動它：
    ```bash
    $ java -javaagent:path/to/opentelemetry-javaagent.jar -jar myapp.jar
    ```

## 配置代理

代理是高度可配置的。

一種選擇是通過 `-D` 旗標來傳遞配置屬性。在此示例中，配置了跟踪的服務名稱和 zipkin 導出器：

```bash
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.service.name=your-service-name \
     -Dotel.traces.exporter=zipkin \
     -jar myapp.jar
```

您還可以使用環境變量來配置代理：

```bash
OTEL_SERVICE_NAME=your-service-name \
OTEL_TRACES_EXPORTER=zipkin \
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -jar myapp.jar
```

您還可以提供 Java 屬性文件並從那裡加載配置值：

```bash
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.javaagent.configuration-file=path/to/properties/file.properties \
     -jar myapp.jar
```

或是:

```bash
OTEL_JAVAAGENT_CONFIGURATION_FILE=path/to/properties/file.properties \
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -jar myapp.jar
```

要查看全部配置選項，請參閱代理配置。

### Exporters

以下配置屬性對所有導出器都是通用的：

|系統屬性|環境變量|目的|
|-------|-------|----|
|otel.traces.exporter	|OTEL_TRACES_EXPORTER	|用於 tracing 的導出器列表，以逗號分隔。默認為 `otlp`。 `none` 表示沒有自動配置的導出器。|
|otel.metrics.exporter	|OTEL_METRICS_EXPORTER	|用於 metrics 的導出器列表，以逗號分隔。默認為 `otlp`。 `none` 表示沒有自動配置的導出器。|
|otel.logs.exporter	|OTEL_LOGS_EXPORTER	|用於記錄的導出器列表，以逗號分隔。默認為`none`。|

#### OTLP exporter (span, metric, and log exporters)

OpenTelemetry 協議 (OTLP) span、metric 和 log 導出器:

|系統屬性|環境變量|目的|
|-------|-------|----|
|otel.traces.exporter=otlp (default)	|OTEL_TRACES_EXPORTER=otlp	|Select the OpenTelemetry exporter for tracing (default)|
|otel.metrics.exporter=otlp (default)	|OTEL_METRICS_EXPORTER=otlp	|Select the OpenTelemetry exporter for metrics (default)|
|otel.logs.exporter=otlp	|OTEL_LOGS_EXPORTER=otlp	|Select the OpenTelemetry exporter for logs|
|otel.exporter.otlp.endpoint	|OTEL_EXPORTER_OTLP_ENDPOINT	|The OTLP traces, metrics, and logs endpoint to connect to. Must be a URL with a scheme of either http or https based on the use of TLS. If protocol is http/protobuf the version and signal will be appended to the path (e.g. v1/traces, v1/metrics, or v1/logs). Default is http://localhost:4317 when protocol is grpc, and http://localhost:4318/v1/{signal} when protocol is http/protobuf.|
|otel.exporter.otlp.traces.endpoint	|OTEL_EXPORTER_OTLP_TRACES_ENDPOINT	|The OTLP traces endpoint to connect to. Must be a URL with a scheme of either http or https based on the use of TLS. Default is http://localhost:4317 when protocol is grpc, and http://localhost:4318/v1/traces when protocol is http/protobuf.|
|otel.exporter.otlp.metrics.endpoint	|OTEL_EXPORTER_OTLP_METRICS_ENDPOINT	|The OTLP metrics endpoint to connect to. Must be a URL with a scheme of either http or https based on the use of TLS. Default is http://localhost:4317 when protocol is grpc, and http://localhost:4318/v1/metrics when protocol is http/protobuf.|
|otel.exporter.otlp.logs.endpoint	|OTEL_EXPORTER_OTLP_LOGS_ENDPOINT	|The OTLP logs endpoint to connect to. Must be a URL with a scheme of either http or https based on the use of TLS. Default is http://localhost:4317 when protocol is grpc, and http://localhost:4318/v1/logs when protocol is http/protobuf.|
|otel.exporter.otlp.certificate	|OTEL_EXPORTER_OTLP_CERTIFICATE	|The path to the file containing trusted certificates to use when verifying an OTLP trace, metric, or log server's TLS credentials. The file should contain one or more X.509 certificates in PEM format. By default the host platform's trusted root certificates are used.|
|otel.exporter.otlp.traces.certificate	|OTEL_EXPORTER_OTLP_TRACES_CERTIFICATE	|The path to the file containing trusted certificates to use when verifying an OTLP trace server's TLS credentials. The file should contain one or more X.509 certificates in PEM format. By default the host platform's trusted root certificates are used.|
|otel.exporter.otlp.metrics.certificate	|OTEL_EXPORTER_OTLP_METRICS_CERTIFICATE	|The path to the file containing trusted certificates to use when verifying an OTLP metric server's TLS credentials. The file should contain one or more X.509 certificates in PEM format. By default the host platform's trusted root certificates are used.|
|otel.exporter.otlp.logs.certificate	|OTEL_EXPORTER_OTLP_LOGS_CERTIFICATE	|The path to the file containing trusted certificates to use when verifying an OTLP log server's TLS credentials. The file should contain one or more X.509 certificates in PEM format. By default the host platform's trusted root certificates are used.|
|otel.exporter.otlp.client.key	|OTEL_EXPORTER_OTLP_CLIENT_KEY	|The path to the file containing private client key to use when verifying an OTLP trace, metric, or log client's TLS credentials. The file should contain one private key PKCS8 PEM format. By default no client key is used.|
|otel.exporter.otlp.traces.client.key	|OTEL_EXPORTER_OTLP_TRACES_CLIENT_KEY	|The path to the file containing private client key to use when verifying an OTLP trace client's TLS credentials. The file should contain one private key PKCS8 PEM format. By default no client key file is used.|
|otel.exporter.otlp.metrics.client.key	|OTEL_EXPORTER_OTLP_METRICS_CLIENT_KEY	|The path to the file containing private client key to use when verifying an OTLP metric client's TLS credentials. The file should contain one private key PKCS8 PEM format. By default no client key file is used.|
|otel.exporter.otlp.logs.client.key	|OTEL_EXPORTER_OTLP_LOGS_CLIENT_KEY|	The path to the file containing private client key to use when verifying an OTLP log client's TLS credentials. The file should contain one private key PKCS8 PEM format. By default no client key file is used.|
|otel.exporter.otlp.client.certificate	|OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE	|The path to the file containing trusted certificates to use when verifying an OTLP trace, metric, or log client's TLS credentials. The file should contain one or more X.509 certificates in PEM format. By default no chain file is used.|
|otel.exporter.otlp.traces.client.certificate	|OTEL_EXPORTER_OTLP_TRACES_CLIENT_CERTIFICATE	|The path to the file containing trusted certificates to use when verifying an OTLP trace server's TLS credentials. The file should contain one or more X.509 certificates in PEM format. By default no chain file is used.|
|otel.exporter.otlp.metrics.client.certificate	|OTEL_EXPORTER_OTLP_METRICS_CLIENT_CERTIFICATE	|The path to the file containing trusted certificates to use when verifying an OTLP metric server's TLS credentials. The file should contain one or more X.509 certificates in PEM format. By default no chain file is used.|
|otel.exporter.otlp.logs.client.certificate	|OTEL_EXPORTER_OTLP_LOGS_CLIENT_CERTIFICATE	|The path to the file containing trusted certificates to use when verifying an OTLP log server's TLS credentials. The file should contain one or more X.509 certificates in PEM format. By default no chain file is used.|
|otel.exporter.otlp.headers	|OTEL_EXPORTER_OTLP_HEADERS	|Key-value pairs separated by commas to pass as request headers on OTLP trace, metric, and log requests.|
|otel.exporter.otlp.traces.headers	|OTEL_EXPORTER_OTLP_TRACES_HEADERS	|Key-value pairs separated by commas to pass as request headers on OTLP trace requests.|
|otel.exporter.otlp.metrics.headers	|OTEL_EXPORTER_OTLP_METRICS_HEADERS	|Key-value pairs separated by commas to pass as request headers on OTLP metrics requests.
|otel.exporter.otlp.logs.headers	|OTEL_EXPORTER_OTLP_LOGS_HEADERS	Key-value pairs separated by commas to pass as request headers on OTLP logs requests.|
|otel.exporter.otlp.compression	|OTEL_EXPORTER_OTLP_COMPRESSION	|The compression type to use on OTLP trace, metric, and log requests. Options include gzip. By default no compression will be used.|
|otel.exporter.otlp.traces.compression	|OTEL_EXPORTER_OTLP_TRACES_COMPRESSION	|The compression type to use on OTLP trace requests. Options include gzip. By default no compression will be used.|
|otel.exporter.otlp.metrics.compression	|OTEL_EXPORTER_OTLP_METRICS_COMPRESSION	|The compression type to use on OTLP metric requests. Options include gzip. By default no compression will be used.|
|otel.exporter.otlp.logs.compression	|OTEL_EXPORTER_OTLP_LOGS_COMPRESSION	|The compression type to use on OTLP log requests. Options include gzip. By default no compression will be used.|
|otel.exporter.otlp.timeout	|OTEL_EXPORTER_OTLP_TIMEOUT	|The maximum waiting time, in milliseconds, allowed to send each OTLP trace, metric, and log batch. Default is 10000.|
|otel.exporter.otlp.traces.timeout	|OTEL_EXPORTER_OTLP_TRACES_TIMEOUT	|The maximum waiting time, in milliseconds, allowed to send each OTLP trace batch. Default is 10000.|
|otel.exporter.otlp.metrics.timeout	|OTEL_EXPORTER_OTLP_METRICS_TIMEOUT	|The maximum waiting time, in milliseconds, allowed to send each OTLP metric batch. Default is 10000.|
|otel.exporter.otlp.logs.timeout	|OTEL_EXPORTER_OTLP_LOGS_TIMEOUT	|The maximum waiting time, in milliseconds, allowed to send each OTLP log batch. Default is 10000.|
|otel.exporter.otlp.protocol	|OTEL_EXPORTER_OTLP_PROTOCOL	|The transport protocol to use on OTLP trace, metric, and log requests. Options include grpc and http/protobuf. Default is grpc.|
|otel.exporter.otlp.traces.protocol	|OTEL_EXPORTER_OTLP_TRACES_PROTOCOL	|The transport protocol to use on OTLP trace requests. Options include grpc and http/protobuf. Default is grpc.|
|otel.exporter.otlp.metrics.protocol	|OTEL_EXPORTER_OTLP_METRICS_PROTOCOL	|The transport protocol to use on OTLP metric requests. Options include grpc and http/protobuf. Default is grpc.|
|otel.exporter.otlp.logs.protocol	|OTEL_EXPORTER_OTLP_LOGS_PROTOCOL	|The transport protocol to use on OTLP log requests. Options include grpc and http/protobuf. Default is grpc.|
|otel.exporter.otlp.metrics.temporality.preference	|OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE	|The preferred output aggregation temporality. Options include DELTA and CUMULATIVE. If CUMULATIVE, all instruments will have cumulative temporality. If DELTA, counter (sync and async) and histograms will be delta, up down counters (sync and async) will be cumulative. Default is CUMULATIVE.|
|otel.exporter.otlp.metrics.default.histogram.aggregation	|OTEL_EXPORTER_OTLP_METRICS_DEFAULT_HISTOGRAM_AGGREGATION	|The preferred default histogram aggregation. Options include EXPONENTIAL_BUCKET_HISTOGRAM and EXPLICIT_BUCKET_HISTOGRAM. Default is EXPLICIT_BUCKET_HISTOGRAM.|
|otel.experimental.exporter.otlp.retry.enabled	|OTEL_EXPERIMENTAL_EXPORTER_OTLP_RETRY_ENABLED	|If true, enable experimental retry support. Default is false.|

要為 OTLP 導出器配置服務名稱，請將 `service.name` 鍵添加到 OpenTelemetry 資源（見下文），例如 `OTEL_RESOURCE_ATTRIBUTES=service.name=myservice`。

#### Jaeger exporter

Jaeger 導出器。該導出器使用 gRPC 作為其通信協議。

|系統屬性|環境變量|目的|
|-------|-------|----|
|otel.traces.exporter=jaeger	|OTEL_TRACES_EXPORTER=jaeger	|選擇 Jaeger 導出器|
|otel.exporter.jaeger.endpoint	|OTEL_EXPORTER_JAEGER_ENDPOINT	|要連接的 Jaeger gRPC 端點。默認為 `http://localhost:14250`。|
|otel.exporter.jaeger.timeout	|OTEL_EXPORTER_JAEGER_TIMEOUT	|允許發送每批的最大等待時間（以毫秒為單位）。默認值為 `10000`。|

#### Zipkin exporter

Zipkin 導出器。它將 Zipkin 格式的 JSON 發送到指定的 HTTP URL。

|系統屬性|環境變量|目的|
|-------|-------|----|
|otel.traces.exporter=zipkin	|OTEL_TRACES_EXPORTER=zipkin	|選擇 Zipkin 導出器|
|otel.exporter.zipkin.endpoint	|OTEL_EXPORTER_ZIPKIN_ENDPOINT	|要連接的 Zipkin 端點。默認為 `http://localhost:9411/api/v2/spans`。目前僅支持 HTTP。|

#### Prometheus exporter

Prometheus 導出器。

|系統屬性|環境變量|目的|
|-------|-------|----|
|otel.metrics.exporter=prometheus	|OTEL_METRICS_EXPORTER=prometheus	|選擇 Prometheus 導出器|
|otel.exporter.prometheus.port	|OTEL_EXPORTER_PROMETHEUS_PORT	|用於綁定 prometheus metric server 的本地端口。默認值為 `9464`。|
|otel.exporter.prometheus.host	|OTEL_EXPORTER_PROMETHEUS_HOST	|用於綁定 prometheus metric server 的本地地址。默認值為 `0.0.0.0`。|

請注意，這是一個拉式導出器 - 它在本地進程上打開一個服務器，偵聽指定的主機和端口，讓 Prometheus 服務器從中抓取指標數據。

#### Logging exporter

日誌導出器將 span 的名稱及其屬性打印到標準輸出。它主要用於測試和調試。

|系統屬性|環境變量|目的|
|-------|-------|----|
|otel.traces.exporter=logging	|OTEL_TRACES_EXPORTER=logging	|對 tracing 選擇 logging 導出器|
|otel.metrics.exporter=logging	|OTEL_METRICS_EXPORTER=logging	|對 metrics 選擇 logging 導出器|
|otel.logs.exporter=logging	|OTEL_LOGS_EXPORTER=logging	|對 logs 選擇 logging 導出器|

#### Propagator

Propagator 確定使用哪些分佈式跟踪標頭格式，以及使用哪些 baggage 傳播標頭格式。

|系統屬性|環境變量|目的|
|-------|-------|----|
|otel.propagators	|OTEL_PROPAGATORS	|要使用的傳播器。對多個傳播者使用逗號分隔的列表。默認為 tracecontext,baggage (W3C)。|

支持的值有:

- `tracecontext`: W3C Trace Context (add baggage as well to include W3C baggage)
- `baggage`: W3C Baggage
- `b3`: B3 Single
- `b3multi`: B3 Multi
- `jaeger`: Jaeger (includes Jaeger baggage)
- `xray`: AWS X-Ray
- `ottrace`: OT Trace


