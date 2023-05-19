# Parameters

**在組件之間傳遞少量數據**

Parameters 對於在組件之間傳遞少量數據以及當組件創建的數據不代表機器學習工件（如模型、數據集或更複雜的數據類型）時很有用。

使用內置的 Python 類型註釋指定參數輸入和輸出：

```python
from kfp import dsl

@dsl.component
def join_words(word: str, count: int = 10) -> str:
    return ' '.join(word for _ in range(count))
```

KFP 根據下表將 Python 類型註釋映射到 [ML 元數據](https://github.com/google/ml-metadata)中存儲的類型：

|Python object	|KFP type|
|---------------|--------|
|`str`	|string|
|`int`	|number|
|`float`	|number|
|`bool`	|boolean|
|`typing.List` / `list`	|object|
|`typing.Dict` / `dict`	|object|

與普通的 Python 函數一樣，輸入參數可以有預設值，以標準方式表示：`def func(my_string: str = 'default')`:

在背後，{==KFP 對要所有組件傳入和傳出的參數使用 `JSON` 來進行序列化與反序列化==}。

對於所有 Python 組件（`Lightweight Python Components` 和 `Containerized Python Components`），參數序列化和反序列化對用戶是不可見的； KFP 會自動處理此問題。

對於 `Container Components`，傳入參數的反序列化對用戶是不可見的； KFP 自動將輸入傳遞給組件。對於容器組件輸出，{==容器組件中的用戶代碼必須處理輸出參數的序列化==}，如 [Container Components: Create component outputs](https://www.kubeflow.org/docs/components/pipelines/v2/components/container-components#create-component-outputs) 所述。

## Input 參數

使用輸入參數非常容易。只需使用 **資料型別** 和 **可選的預設值** 來註釋您的組件函數。以下管道演示了這一點，該管道使用 Python 組件、容器組件和將所有參數類型作為輸入的管道：

```python
from typing import Dict, List
from kfp import dsl

@dsl.component
def python_comp(
    string: str = 'hello',
    integer: int = 1,
    floating_pt: float = 0.1,
    boolean: bool = True,
    dictionary: Dict = {'key': 'value'},
    array: List = [1, 2, 3],
):
    print(string)
    print(integer)
    print(floating_pt)
    print(boolean)
    print(dictionary)
    print(array)


@dsl.container_component
def container_comp(
    string: str = 'hello',
    integer: int = 1,
    floating_pt: float = 0.1,
    boolean: bool = True,
    dictionary: Dict = {'key': 'value'},
    array: List = [1, 2, 3],
):
    return dsl.ContainerSpec(
        image='alpine',
        command=['sh', '-c', """echo $0 $1 $2 $3 $4 $5 $6"""],
        args=[
            string,
            integer,
            floating_pt,
            boolean,
            dictionary,
            array,
        ])

@dsl.pipeline
def my_pipeline(
    string: str = 'Hey!',
    integer: int = 100,
    floating_pt: float = 0.1,
    boolean: bool = False,
    dictionary: Dict = {'key': 'value'},
    array: List = [1, 2, 3],
):
    python_comp(
        string='howdy',
        integer=integer,
        array=[4, 5, 6],
    )
    container_comp(
        string=string,
        integer=20,
        dictionary={'other key': 'other val'},
        boolean=boolean,
    )
```

## Output 參數

對於 `Python 組件`和管道，輸出參數通過返回註釋指示：

```python
from kfp import dsl

@dsl.component
def my_comp() -> int:
    return 1

@dsl.pipeline
def my_pipeline() -> int:
    task = my_comp()
    return task.output
```

對於 `容器組件`，輸出參數使用 `dsl.OutputPath` 註釋指示：

```python
from kfp import dsl

@dsl.container_component
def my_comp(int_path: dsl.OutputPath(int)):
    return dsl.ContainerSpec(
        image='alpine',
        command=[
            'sh', '-c', f"""mkdir -p $(dirname {int_path})\
                            && echo 1 > {int_path}"""
        ])

@dsl.pipeline
def my_pipeline() -> int:
    task = my_comp()
    return task.outputs['int_path']
```

有關如何使用 `dsl.OutputPath` 的更多信息，請參閱[Container Components: Create component outputs](https://www.kubeflow.org/docs/components/pipelines/v2/components/container-components#create-component-outputs)

## 多個 Output 參數

您可以使用 [`typing.NamedTuple`](https://docs.python.org/3/library/typing.html#typing.NamedTuple) 指定多個命名輸出參數。您可以在 [`PipelineTask`](https://kubeflow-pipelines.readthedocs.io/en/master/source/dsl.html#kfp.dsl.PipelineTask) 上使用 `.output['<output-key>']` 訪問命名輸出：

```python
from kfp import dsl
from typing import NamedTuple

@dsl.component
def my_comp() -> NamedTuple('outputs', a=int, b=str):
    outputs = NamedTuple('outputs', a=int, b=str)
    return outputs(1, 'hello')

@dsl.pipeline
def my_pipeline() -> NamedTuple('pipeline_outputs', c=int, d=str):
    task = my_comp()
    pipeline_outputs = NamedTuple('pipeline_outputs', c=int, d=str)
    return pipeline_outputs(task.outputs['a'], task.outputs['b'])
```



