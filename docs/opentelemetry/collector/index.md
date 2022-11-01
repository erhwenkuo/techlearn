# OTEL Collector

原文: [Collector](https://opentelemetry.io/docs/collector/)

讓接收、處理和導出遙測數據的方式與供應商脫勾的手法。

![](./assets/Otel_Collector.svg)

## 介紹

OpenTelemetry Collector 提供了一個與供應商無關的實現，用於接收、處理和導出遙測數據。它消除了運行、操作和維護多個代理/收集器的需要。 OTEL Collector 改進了它的可擴展性並支持將開源可觀察性數據格式（例如 Jaeger、Prometheus、Fluent Bit 等）發送到一個或多個開源或商業後端。

本地的 Collector agent 是儀器化函式庫導出其遙測數據的默認位置。

## 目標

- **Usability**：合理的默認配置，支持流行的協議，開箱即用。
- **Performance**：在不同的負載和配置下高度穩定和高性能。
- **Observability**：可觀察性服務的範例。
- **Extensibility**：可定制，無需觸及核心代碼。
- **Unification**：單一代碼庫，可部署為 agent 或 collector，支持跟踪、指標和日誌。

## 何時使用 Collector

你可能想知道在什麼情況下使用 collector 發送遙測數據，而不是讓每個服務直接發送到遙測數據後端？

對於試用和開始使用 OpenTelemetry 的專案來說，將數據遙測直接發送到後端(比如Jaeger、Zipkin..etc)是快速獲得價值的好方法。此外，在開發或小規模環境中，您可以在沒有 collector 的情況下獲得不錯的結果。

但是，通常我們建議在您的服務旁邊使用 collector，因為它允許您的服務快速卸載數據，並且 collector 可以幫忙處理一些其他例外的處理任務，例如重試、批次處理、加密甚至過濾敏感數據。

[設置收集器](https://opentelemetry.io/docs/collector/getting-started)也比您想像的要容易：每種語言的默認 OTLP 導出器都假定一個本地收集器端點，因此您將啟動一個收集器，然後就可以開始獲取遙測數據。

## 狀態和發布

收集器組件的成熟度級別不同。OpenTelemetry 正在努力確保每個組件都有其穩定性記錄。要跟踪這項工作的進度，請參閱[ opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/10116)。

