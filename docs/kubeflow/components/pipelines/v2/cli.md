# 命令行界面

**通過 CLI 與 KFP 交互**

本節提供 KFP CLI 中可用命令的摘要。有關 [KFP CLI 中所有可用命令](https://kubeflow-pipelines.readthedocs.io/en/master/source/cli.html)的更全面的文檔，請參閱 [KFP SDK 參考文檔](https://kubeflow-pipelines.readthedocs.io/en/master/index.html)中的命令行界面。

## 用法

KFP CLI 作為 `kfp` 與 KFP SDK 一起安裝。

### 檢查 KFP CLI 的可用性

要檢查您的環境中是否安裝了 KFP CLI，請運行以下命令：

```bash
kfp --version
```

### 一般語法

KFP CLI 中的所有命令都使用以下通用語法：

```bash
kfp [OPTIONS] COMMAND [ARGS]...
```

例如，要列出特定端點的所有運行，請運行以下命令：

```bash
kfp --endpoint http://my_kfp_endpoint.com run list
```

### 獲取命令幫助

要獲取特定命令的幫助，請直接在命令行中使用參數 `--help`。例如，要查看有關 `kfp run` 命令的指南，請鍵入以下命令：

```bash
kfp run --help
```

## KFP CLI 的主要功能

### 與 KFP 資源互動

大多數 KFP CLI 命令允許您從 KFP 後端創建、讀取、更新或刪除 KFP 資源。所有這些命令都使用以下通用語法：

```python
kfp <resource_name> <action>
```

`<resource_name>` 參數可以是以下之一：

- `run`
- `recurring-run`
- `pipeline`
- `experiment`

對於 `<resource_name>` 參數的所有值，`<action>` 參數可以是以下之一：

- `create`
- `list`
- `get`
- `delete`

某些資源名稱具有額外的特定於資源的操作。下表列出了一些特定於資源的操作示例：

| Resource name	|Additional resource-specific actions|
|---------------|------------------------------------|
|run	|-archive<br/>-unarchive|
|recurring-run|-disable<br/>-enable|
|experiment|-archive<br/>-unarchive|
|pipeline|-create-version<br/>-list-versions<br/>-get-versions<br/>-delete-versions|

### 編譯管道

您可以使用 `kfp dsl compile` 命令將 Python 文件中定義的管道或組件編譯為 IR YAML。

1. 要編譯在 Python 文件中定義的管道定義，請運行以下命令。

    ```python
    kfp dsl compile --py [PATH_TO_INPUT_PYTHON] --output [PATH_TO_OUTPUT_YAML] --function [PIPELINE_NAME]
    ```

    例如：

    ```python
    kfp dsl compile --py path/to/pipeline.py --output path/to/output.yaml
    ```

    要從包含多個管道或組件定義的 Python 文件編譯單個管道或組件，請使用 `--function` 參數。

    例如：

    ```python
    kfp dsl compile --py path/to/pipeline.py --output path/to/output.yaml --function my_pipeline
    ```

    ```python
    kfp dsl compile --py path/to/pipeline.py --output path/to/output.yaml --function my_component
    ```

2. 要指定管道參數，請使用 `--pipeline-parameters` 參數並將參數作為 JSON 提供。

    ```python
    kfp dsl compile [PATH_TO_INPUT_PYTHON] --output [PATH_TO_OUTPUT_YAML] --pipeline-parameters [PIPELINE_PARAMETERS_JSON]
    ```

    例如：

    ```python
    kfp dsl compile --py path/to/pipeline.py --output path/to/output.yaml --pipeline-parameters '{"param1": 2.0, "param2": "my_val"}'
    ```

### 構建容器化的 Python 組件

您可以在 KFP SDK 中編寫[容器化 Python 組件](https://www.kubeflow.org/docs/components/pipelines/v2/components/containerized-python-components)。與更簡單的[輕量級 Python 組件](https://www.kubeflow.org/docs/components/pipelines/v2/components/lightweight-python-components)創作體驗相比，這使您可以使用更好的程式碼來組織與設計組件。

#### 在你開始之前

運行以下命令以安裝具有附加 Docker 依賴項的 KFP SDK(要注意 kfp 的版本與 Kubeflow 版本的兼容性)：

```bash
pip install --pre kfp[all]
```

#### 構建組件

要構建容器化 Python 組件，請在 KFP CLI 中使用以下便捷命令。使用此命令，您可以使用在 COMPONENTS_DIRECTORY 中找到的所有源代碼構建一個映像。該命令使用在目錄中找到的組件作為組件運行時入口點。

```bash
kfp component build [OPTIONS] [COMPONENTS_DIRECTORY] [ARGS]...
```

例如：

```bash
kfp component build src/ --component-filepattern my_component --push-image
```

有關 `kfp component build` 命令支持的參數和旗標的更多信息，請參閱 [KFP SDK API 參考](https://kubeflow-pipelines.readthedocs.io/en/master/index.html)中的[構建](https://kubeflow-pipelines.readthedocs.io/en/master/source/cli.html#kfp-component-build)。有關創建容器化 Python 組件的更多信息，請參閱[編寫 Python 容器化組件](https://www.kubeflow.org/docs/components/pipelines/v2/components/containerized-python-components)。