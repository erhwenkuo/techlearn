# Run Trigger

**Kubeflow Pipelines 中運行觸發器的概念概述**

運行觸發器是一個標誌，它告訴系統循環運行配置何時產生新的運行。以下類型的運行觸發器可用：

- Periodic：用於基於間隔的運行計劃（例如：每 2 小時或每 45 分鐘）。
- Cron：用於指定計劃運行的 `cron` 語義。