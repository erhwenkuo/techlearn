# 用於 Python 的 OpenTelemetry 跟踪 API

原文: [](https://uptrace.dev/opentelemetry/python-tracing.html)

![OpenTelemetry Tracing API for Python](./assets/python-original.svg)

## 安裝

[OpenTelemetry-Python](https://github.com/open-telemetry/opentelemetry-python) 是 OpenTelemetry 的 Python 實現。它提供了 OpenTelemetry Tracing API，您可以使用它來通過 [OpenTelemetry tracing](https://uptrace.dev/opentelemetry/distributed-tracing.html) 來檢測您的應用程序。

```bash
pip install opentelemetry-api
```

## 快速開始

**Step 1.** 讓我們檢測以下函數:

```python
def insert_user(**kwargs):
    return User.objects.create_user(**kwargs)
```

**Step 2.** 用 span 包裝操作:

```python hl_lines="1 3 6"
from opentelemetry import trace

tracer = trace.get_tracer("app_or_package_name", "1.0.0")

def insert_user(**kwargs):
    with tracer.start_as_current_span("insert-user") as span:
        return User.objects.create_user(**kwargs)
```

**Step 3.** 用屬性(attributes)記錄上下文信息:

```python hl_lines="3-5"
def insert_user(**kwargs):
    with tracer.start_as_current_span("insert-user") as span:
        if span.is_recording():
            span.set_attribute("enduser.id", kwargs["id"])
            span.set_attribute("enduser.email", kwargs["email"])
        return User.objects.create_user(**kwargs)
```

就是這樣！您不必手動記錄異常，因為 `tracer.start_as_current_span` 會自動為您完成。


### Tracer

要開始創建 span，您需要一個 tracer 實例。您可以通過提供執行檢測的庫/應用程序的名稱和版本來創建 tracer:

```python hl_lines="3"
from opentelemetry import trace

tracer = trace.get_tracer("app_or_package_name", "1.0.0")
```

您可以擁有任意數量的 tracer，但通常您只需要一個應用程序或庫的 tracer。稍後，您可以使用 tracer 名稱來標識生成跨度的 instrumentation。

### 創建 spans

一旦有了 tracer，創建 span 就很容易了:

```python
# Create a span with name "operation-name" and kind="server".
with tracer.start_as_current_span("operation-name", kind=trace.SpanKind.SERVER) as span:
    do_some_work()
```

### 添加 span attributes

要記錄上下文信息，您可以使用屬性註釋 span。例如，HTTP 端點可能具有 `http.method = GET` 和 `http.route = /projects/:id` 這樣的屬性。

```python
# To avoid expensive computations, check that span is recording
# before setting any attributes.
if span.is_recording():
    span.set_attribute("http.method", "GET")
    span.set_attribute("http.route", "/projects/:id")
```

您可以根據需要命名 attribute，但對於常見 operation，您應該使用 [semantic attributes](https://uptrace.dev/opentelemetry/attributes.html)。

### 添加 span events

您可以使用 event 註釋 span，例如，您可以使用 event 來記錄日誌消息:

```python
span.add_event("log", {
    "log.severity": "error",
    "log.message": "User not found",
    "enduser.id": "123",
})
```

### 設置 status code

您可以設置錯誤狀態代碼以指示 operation 包含錯誤:

```python
except ValueError as exc:
    span.set_status(trace.Status(trace.StatusCode.ERROR, str(exc)))
```

### 記錄 exceptions

OpenTelemetry 提供了記錄異常的快捷方式，通常與 `set_status` 一起使用:

```python
except ValueError as exc:
    # Record the exception and update the span status.
    span.record_exception(exc)
    span.set_status(trace.Status(trace.StatusCode.ERROR, str(exc)))
```

### Context

OpenTelemetry 將 active span 存儲在上下文中，並將上下文保存在 thread-local 存儲中。您可以將上下文相互嵌套，OpenTelemetry 將在您結束跨度時自動激活 parent span context。

`start_as_current_span` 為您設置 active span，但您也可以手動 activate span:

```python
# Activate the span in the current context.
with trace.use_span(span, end_on_exit=True):
    do_some_work()
```

要從當前上下文中獲取 active span:

```python
span = trace.get_current_span()
```

如果您使用的是分佈式服務，則可以使用請求檢測將上下文傳播到另一個服務。

## Auto Instrumentation

opentelemetry-python 允許您使用 `opentelemetry-instrumentation` 套件中的 `opentelemetry-instrument` utility 自動檢測任何 Python 應用程序。

首先，您需要安裝 `opentelemetry-instrument` 可執行文件：

```python
pip install opentelemetry-instrumentation
```

然後，安裝應該由 `opentelemetry-instrument` 自動應用的檢測：

```python
pip install opentelemetry-instrumentation-flask
```

並使用 `opentelemetry-instrument` 包裝器運行您的應用程序：

```python
opentelemetry-instrument python3 main.py
```

有關詳細信息，請參閱 [flask-auto-instrumentation](https://github.com/uptrace/uptrace-python/tree/master/example/flask-auto-instrumentation) 範例。


