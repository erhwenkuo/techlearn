# 載入與分享組件

**加載和使用組件生態系統**

本節介紹如何加載和使用現有組件。本節中的“組件”既指單步組件，也指流水線，也可以[作為組件使用](https://www.kubeflow.org/docs/components/pipelines/v2/pipelines/pipeline-basics#pipelines-as-components)。

IR YAML 用作可移植、可共享的計算模板。這允許您編譯組件並與他人共享，以及利用現有組件的生態系統。

要使用現有組件，您可以使用 `components` 模塊加載它，並將它與管道中的其他組件一起使用：

```python
from kfp import components

loaded_comp = components.load_component_from_file('component.yaml')

@dsl.pipeline
def my_pipeline():
    loaded_comp()
```

您還可以直接從 URL 加載組件，例如 GitHub URL：

```python
loaded_comp = components.load_component_from_url('https://github.com/kubeflow/pipelines/blob/984d8a039d2ff105ca6b21ab26be057b9552b51d/sdk/python/test_data/pipelines/two_step_pipeline.yaml')
```

最後，您可以使用 `components.load_component_from_text` 從字符串加載組件：

```python
with open('component.yaml') as f:
    component_str = f.read()

loaded_comp = components.load_component_from_text(component_str)
```

一些函式庫，例如 [Google Cloud Pipeline Components](https://cloud.google.com/vertex-ai/docs/pipelines/components-introduction) 打包並在 pip 可安裝的 [Python 包](https://pypi.org/project/google-cloud-pipeline-components/)中提供可重用的組件。

