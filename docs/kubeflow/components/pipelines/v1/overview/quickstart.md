# 快速開始

如果您想了解 Kubeflow Piplines 用戶界面 (UI) 並且快速運行簡單的管道，請使用本指南。

本快速入門指南的目的是展示如何使用 Kubeflow Pipelines 安裝時所附帶的兩個範例，這些範例在 Kubeflow Pipelines UI 上可查看到。您可以將本指南用作 Kubeflow Pipelines UI 的介紹。

## 部署 Kubeflow 並打開 Kubeflow Pipelines UI

有多種部署 [Kubeflow Pipelines 的選項](https://www.kubeflow.org/docs/components/pipelines/installation/overview/)，請選擇最適合您需求的選項。如果您不確定並且只想嘗試 kubeflow 管道，建議從[獨立部署](https://www.kubeflow.org/docs/components/pipelines/installation/standalone-deployment/)開始。

部署 Kubeflow Pipelines 後，請確保您可以訪問 UI。訪問 UI 的步驟因您用於部署 Kubeflow 管道的方法而異。

## 運行 Basic 管道

Kubeflow Pipelines 提供了一些示例，您可以使用它們快速試用 Kubeflow Pipelines。以下步驟向您展示瞭如何運行包含一些 Python 操作但不包含機器學習 (ML) 工作負載的基本示例：

1. 在管道 UI 上點擊範例名稱，`[Tutorial] Data passing in python components`：

    ![](./assets/v1-pipeline-tutorial.png)

2. 點擊 "Create experiment"：

    ![](./assets/v1-pipeline-tutorial-experiment.png)

3. 按照提示創建 experiment，然後創建 run。該示例為您需要的所有參數提供默認值。以下屏幕截圖假設您已經創建了一個名為 `My first experiment` 的實驗，現在正在創建一個名為 `My first run` 的運行：

    ![](./assets/v1-pipeline-tutorial-experiment-create.png)

    ![](./assets/v1-pipeline-tutorial-experiment-run.png)

4. 點擊 **Start** 以運行管道。

5. 點擊 experiment 儀表板上的 **run** 名稱：

    ![](./assets/v1-pipeline-tutorial-experiment-run-view.png)

6. 通過點擊圖形的組件和其他 UI 元素來探索圖形和 run 的其他方面資訊：

    ![](./assets/v1-pipeline-tutorial-experiment-run-detail.png)

您可以在 [Kubeflow Pipelines 存儲庫](https://github.com/kubeflow/pipelines/tree/sdk/release-1.8/samples/tutorials/Data%20passing%20in%20python%20components)中找到 `Data passing in python components tutorial` 的源碼。

