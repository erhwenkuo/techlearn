# 容器化 Python 組件

以下假定您基本熟悉 [輕量級 Python 組件](./lightweight-python-components.md) 的相關知識與原理。

**容器化 Python 組件** 通過放寬輕量級 Python 組件是密封的（即完全獨立的）限制來擴展輕量級 Python 組件。這意味著容器化 Python 組件函數可以依賴於函數外部定義的符號、函數外部的導入、相鄰 Python 模塊中的代碼等。為實現這一點，KFP SDK 提供了一種將 Python 代碼打包到容器中的便捷方法。

下面展示如何通過修改 **輕量級 Python 組件** 範例來轉換成 **容器化 Python 組件** ：

```python
from kfp import dsl

@dsl.component
def add(a: int, b: int) -> int:
    return a + b
```

## 1. 源碼設置

首先創建一個空的 `src/` 目錄來包含您的源碼：

```bash
src/
```

接下來，添加以下帶有一個輔助函數的簡單模塊 `src/math_utils.py`：

```python title="src/math_utils.py"
# src/math_utils.py
def add_numbers(num1, num2):
    return num1 + num2
```

最後，將您的組件移動到 `src/my_component.py` 並修改它以使用輔助函數：

```python
# src/my_component.py
from kfp import dsl
from math_utils import add_numbers

@dsl.component
def add(a: int, b: int) -> int:
    return add_numbers(a, b)
```

`src/` 目錄現在看起來像這樣：

```bash
src/
├── my_component.py
└── math_utils.py
```


## 2.修改 dsl.component 裝飾器

在此步驟中，您將向 `src/my_component.py` 中組件的 `@dsl.component` 裝飾器提供 `base_image` 和 `target_image` 參數：

```python
@dsl.component(base_image='python:3.7',
               target_image='gcr.io/my-project/my-component:v1')
def add(a: int, b: int) -> int:
    return add_numbers(a, b)
```

設置 `target_image` 有下列兩個意義

- (a) 指定您將在步驟 3 中構建的鏡像的標籤
- (b) 指示 KFP 在使用帶有該標籤的鏡像的容器中運行修飾的 Python 函數

在容器化 Python 組件中，`base_image` 指定 KFP 在構建新容器鏡像時將使用的基礎鏡像。具體來說，KFP 在用於構建映像的 Dockerfile 中使用 FROM 指令的 `base_image` 參數。

為清楚起見，前面的示例包括 `base_image`，但這不是必需的，因為如果省略，`base_image` 將默認為 `python:3.7`。

## 3.構建組件

現在您的代碼位於獨立目錄中並且您已經指定了 `target_image`，您可以使用 `kfp component build` CLI 命令方便地構建鏡像：

```bash
kfp component build src/ --component-filepattern my_component.py --no-push-image
```

如果您已將 Docker 配置為使用私有 container registry 服務，則可以將 `--no-push-image` 標誌替換為 `--push-image` 以在構建後自動推送鏡像。

## 4.在管道中使用組件

最後，您可以在管道中使用該組件：

```python
from kfp import dsl

@dsl.pipeline
def addition_pipeline(x: int, y: int) -> int:
    task1 = add(a=x, b=y)
    task2 = add(a=task1.output, b=x)
    return task2.output

compiler.Compiler().compile(addition_pipeline, 'pipeline.yaml')
```
