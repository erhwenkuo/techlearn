# 特定用例：Importer 組件

**從管道外部導入組件**

與其他三種 Pipeline 組件的創作方法不同，importer component 不是一般組件的創作風格，而是針對特定用例的預構建的組件：將機器學習 pipeline 的某種工件(artifact)從 URI 加載到當前管道中，並因此加載到 [ML 元數據](https://github.com/google/ml-metadata)中.本節假定您熟悉 KFP [預建的組件 artifacts](https://www.kubeflow.org/docs/components/pipelines/v2/data-types/artifacts)。

如 [Pipeline Basics](https://www.kubeflow.org/docs/components/pipelines/v2/pipelines/pipeline-basics) 中所述，任務的輸入通常是上游任務的輸出。在這種情況下，可以使用 `my_task.outputs['<output-key>']` 在上游任務上輕鬆訪問工件。當上游任務創建工件時，它也會在 [ML 元數據](https://github.com/google/ml-metadata)中註冊。

如果您希望使用不是由當前管道中的任務生成的現有工件，或者希望將根本不是由管道生成的外部文件用作工件，您可以使用 `dsl.importer` 組件加載來自其 URI 的工件。

您不需要編寫導入器組件；可以從 `dsl` 模塊導入直接使用：

```python
from kfp import dsl

@dsl.pipeline
def my_pipeline():
    task = get_date_string()
    importer_task = dsl.importer(
        artifact_uri='gs://ml-pipeline-playground/shakespeare1.txt',
        artifact_class=dsl.Dataset,
        reimport=True,
        metadata={'date': task.output})
    other_component(dataset=importer_task.output)
```

除了 `artifact_uri` 參數之外，您還必須提供 `artifact_class` 參數來指定工件的類型。

導入器組件允許通過元數據參數設置工件元數據。可以使用上游任務的輸出構建元數據，就像示例管道中的 “date” 值一樣。

您還可以指定一個 `reimport` 參數。如果 `reimport` 為 `False`，KFP 將檢查工件是否已導入 ML 元數據，如果已導入，則使用它。當多個管道運行導入相同的工件時，這對於避免 ML 元數據中的重複工件條目很有用。如果 `reimport` 為 `True`，KFP 將重新導入工件作為 ML 元數據中的新工件，而不管它之前是否導入過。

