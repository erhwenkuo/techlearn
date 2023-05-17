# 容器組件

**通過任意容器定義創建組件**

在 KFP 中，每個任務執行會對應在一個容器裡來執行。這意味著所有組件，甚至 Python 組件，都是由 `image`、`command` 和 `args` 定義的。

Python 組件是獨一無二的，因為它們將容器定義抽象出來，使得構建使用純 Python 的組件變得很方便。在背後，KFP SDK 將`image`、`command` 和 `args` 設置為為用戶執行 Python 組件所需的值。

與 Python 組件不同，**容器組件** 允許組件作者直接設計鏡像、命令和參數。這使得編寫執行 shell 腳本、或使用其他語言和二進製文件等的組件成為可能，所有功能的整合都來自 KFP Python SDK。

## 一個簡單的容器組件

下面從一個簡單的 `say_hello` 容器組件開始，逐漸修改它，直到它等同於我們在 `Hello World Pipeline` 示例中的 `say_hello` 組件。這是一個簡單的容器組件：

```python
from kfp import dsl

@dsl.container_component
def say_hello():
    return dsl.ContainerSpec(image='alpine', command=['echo'], args=['Hello'])
```

要創建容器組件，請使用 `dsl.container_component` 裝飾器並創建一個返回 `dsl.ContainerSpec` 對象的函數。 `dsl.ContainerSpec` 接受三個參數：`image`、`command` 和 `args`。上面的組件在運行鏡像 alpine 的容器中運行帶有參數 Hello 的命令 echo。

容器組件可以像 Python 組件一樣在管道中使用：

```python
from kfp import dsl
from kfp import compiler

@dsl.pipeline
def hello_pipeline():
    say_hello()

compiler.Compiler().compile(hello_pipeline, 'pipeline.yaml')
```

如果運行此管道，您將在 `say_hello` 的日誌中看到字符串 Hello。

## 使用組件輸入

為了讓容器組件更容易被使用，`say_hello` 應該能夠接受參數。您可以修改 `say_hello` 以便它接受輸入參數名稱：

```python
from kfp import dsl

@dsl.container_component
def say_hello(name: str):
    return dsl.ContainerSpec(image='alpine', command=['echo'], args=[f'Hello, {name}!'])
```

Container Component 函數中的 **參數** 和 **註解** 聲明了組件的接口。在這種情況下，組件有一個輸入參數名稱，沒有輸出參數。

編譯此組件時，`name` 將替換為佔位符。在運行時，此佔位符將替換為提供給 `say_hello` 組件的名稱的實際值。

實現此組件的另一種方法是使用 `sh -c` 從單個字符串中讀取命令並將名稱作為參數傳遞。這種方法往往更靈活，因為它很容易允許將多個命令鏈接在一起。

```python
from kfp import dsl

@dsl.container_component
def say_hello(name: str):
    return dsl.ContainerSpec(image='alpine', command=['sh', '-c', 'echo Hello, $0!'], args=[name])
```

當您使用參數 `name='World'` 運行組件時，您將看到字符串 `Hello, World!` 在 say_hello 的日誌中。

## 創建組件輸出

與 Python 函數組件不同，容器組件沒有返回值的標準機制。為了使容器組件具有輸出，{==KFP 要求您將輸出寫入容器內某種檔案結構的文件==}。 KFP 將讀取此文件並保存這個文件做為此組件的輸出。

要從 say_hello 組件返回輸出字符串，您可以使用 `dsl.OutputPath(str)` 註釋向函數添加輸出參數：

```python
@dsl.container_component
def say_hello(name: str, greeting: dsl.OutputPath(str)):
    ...
```

該組件現在有一個名為 `name` 型別為 `str` 的輸入參數和一個名為 `greeting` 也為 `str` 類型的輸出參數。在運行時，使用 `dsl.OutputPath` 註釋的參數將提供系統生成的路徑作為參數。{==您的組件邏輯應將輸出值作為 JSON 寫入此路徑==}。 `greeting: dsl.OutputPath(str)` 中的參數 str 描述了輸出問候語的類型（例如，寫入路徑問候語的 JSON 將是一個字符串）。您可以填寫命令和參數以寫入輸出：

```python
@dsl.container_component
def say_hello(name: str, greeting: dsl.OutputPath(str)):
    """Log a greeting and return it as an output."""

    return dsl.ContainerSpec(
        image='alpine',
        command=[
            'sh', '-c', '''RESPONSE="Hello, $0!"\
                            && echo $RESPONSE\
                            && mkdir -p $(dirname $1)\
                            && echo $RESPONSE > $1
                            '''
        ],
        args=[name, greeting])
```

## 在管道中使用

最後，您可以在管道中使用更新後的 `say_hello` 組件：

```python
from kfp import dsl
from kfp import compiler

@dsl.pipeline
def hello_pipeline(person_to_greet: str) -> str:
    # greeting argument is provided automatically at runtime!
    hello_task = say_hello(name=person_to_greet)
    return hello_task.outputs['greeting']

compiler.Compiler().compile(hello_pipeline, 'pipeline.yaml')
```

請注意，在構建管道時，您不需要向組件提供輸出參數；輸出參數總是在運行時由後端自動提供。

這應該看起來與 [Hello World 管道](https://www.kubeflow.org/docs/components/pipelines/v2/hello-world)非常相似，但有一個關鍵區別：{==由於 `greeting` 是一個命名的輸出參數，我們使用 `hello_task.outputs['greeting']` 訪問它並從管道返回它，而不是 `hello_task.output`==}。數據傳遞在 [Pipelines Basics](https://www.kubeflow.org/docs/components/pipelines/v2/pipelines/pipeline-basics) 中有更詳細的討論。

## 特殊佔位符

三種組件創作風格中的每一種都會自動處理通過容器命令和參數中的佔位符傳遞到組件中的數據。通常，您不需要知道它是如何工作的。如果您願意，容器組件還使您能夠直接使用兩個特殊的佔位符：[`dsl.ConcatPlaceholder`](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.ConcatPlaceholder) 和 [`dsl.IfPresentPlaceholder`](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.IfPresentPlaceholder)。

您只能在通過 [`@dsl.container_component`](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.container_component) 裝飾器編寫的容器組件返回的 [`dsl.ContainerSpec`](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.ContainerSpec) 中使用這些佔位符。

### dsl.ConcatPlaceholder

當您提供容器 `command` 或容器 `args` 作為字符串列表時，列表中的每個元素都使用空格分隔符連接起來，然後在運行時發送給容器。在沒有空格分隔符的情況下將一個輸入連接到另一個字符串需要 [`dsl.ConcatPlaceholder`](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.ConcatPlaceholder) 提供的特殊處理。

`dsl.ConcatPlaceholder` 採用一個參數 `items`，它可以是靜態字符串、上游輸出、管道參數或 `dsl.ConcatPlaceholder` 或 `dsl.IfPresentPlaceholder` 的其他實例的任意組合的列表。在運行時，這些字符串將在沒有分隔符的情況下連接在一起。

例如，您可以使用 `dsl.ConcatPlaceholder` 連接文件路徑前綴、後綴和擴展名：

```python
from kfp import dsl

@dsl.container_component
def concatenator(prefix: str, suffix: str):
    return dsl.ContainerSpec(
        image='alpine',
        command=[
            'my_program.sh'
        ],
        args=['--input', dsl.ConcatPlaceholder([prefix, suffix, '.txt'])]
    )
```

### dsl.IfPresentPlaceholder

[`dsl.IfPresentPlaceholder`](https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.IfPresentPlaceholder) 用於有條件地提供命令行參數。 `dsl.IfPresentPlaceholder` 採用三個參數：`input_name`、`then` 和可選的 `else_`。這個佔位符最容易通過一個例子來理解：

```python
@dsl.container_component
def hello_someone(optional_name: str = None):
    return dsl.ContainerSpec(
        image='python:3.7',
        command=[
            'say_hello',
            dsl.IfPresentPlaceholder(
                input_name='optional_name', then=['--name', optional_name])
        ])
```

如果 `hello_someone` 組件作為 `optional_name` 的參數傳遞 world，該組件會將 `--name world` 傳遞給可執行文件 `say_hello`。如果未提供 `optional_name`，則省略 `--name world`。

第三個參數 `else_` 可用於提供默認值，以便在未提供 `input_name` 時回退。例如：

```python
@dsl.container_component
def hello_someone(optional_name: str = None):
    return dsl.ContainerSpec(
        image='python:3.7',
        command=[
            'say_hello',
            dsl.IfPresentPlaceholder(
                input_name='optional_name',
                then=['--name', optional_name],
                else_=['--name', 'friend'])
        ])
```

`then` 和 `else_` 的參數可以是靜態字符串、上游輸出、管道參數或 `dsl.ConcatPlaceholder` 或 `dsl.IfPresentPlaceholder` 的其他實例的任意組合的列表

