# 輕量級 Python 組件

開始編寫組件的最簡單方法是創建 **輕量級 Python 組件**。我們在 [Hello World 管道範例](../hello-world.md)中看到了一個帶有 `say_hello` 的輕量級 Python 組件範例。這是另一個將兩個整數相加的輕量級 Python 組件：

```python
from kfp import dsl

@dsl.component
def add(a: int, b: int) -> int:
    return a + b
```

輕量級 Python 組件是通過使用 `@dsl.component` 裝飾器裝飾 {==Python 函數==}來構建的。 `@dsl.component` 裝飾器將您的函數轉換為 KFP 組件，該組件可以由 KFP 一致性後端作為遠程函數執行，可以獨立執行，也可以作為更複雜管道中的單個步驟執行。

## Python 函數要求

要使用 `@dsl.component` 裝飾器裝飾函數，它必須滿足兩個要求：

1. **Type annotations**：函數輸入和輸出必須具有有效的 [KFP 資料型別註釋](../data-types/index.md)。

    KFP 中有兩類輸入和輸出：`parameter` 和 `artifact`。每個類別中都有特定類型的參數和工件。每個輸入和輸出都將具有由其類型註釋指示的特定類型。

    在前面的 `add` 組件中，輸入 `a` 和 `b` 都是類型為 `int` 的參數。有一個輸出，類型也是 `int`。

    有效的參數註釋包括 Python 內置的 `int`、`float`、`str`、`bool`、`typing.Dict` 和 `typing.List`。[Data Types: Artifacts](../data-types/artifacts.md) 中詳細討論了對工件註釋。

2. **Hermetic**：Python 函數不能引用在其函數體之外定義的任何 symbols。

    例如，如果您希望使用常量，則必須在函數內部定義常量：

    ```python
    @dsl.component
    def double(a: int) -> int:
        """Succeeds at runtime."""
        VALID_CONSTANT = 2
        return VALID_CONSTANT * a
    ```

    相比之下，以下是無效元件並且會在運行時失敗：

    ```python
    # non-example!
    INVALID_CONSTANT = 2

    @dsl.component
    def errored_double(a: int) -> int:
        """Fails at runtime."""
        return INVALID_CONSTANT * a
    ```

    Imports 也必須包含在函數體中：

    ```python
    @dsl.component
    def print_env():
        import os
        print(os.environ)
    ```

    對於許多組件的實現，氣密性(hermeticism)可能是一個相當嚴格的要求。[Containerized Python Components](https://www.kubeflow.org/docs/components/pipelines/v2/components/containerized-python-components) 是另一種更靈活的組件創作方法，可以降低對此氣密性 (hermeticism) 的要求。

## dsl.component 裝飾參數

在上面的示例中，我們使用了只有一個參數的 `@dsl.component` 裝飾器來註釋 Python 函數。裝飾器也可接受一些額外的參數。

### packages_to_install

實務上, 輕量級 Python 組件一般來說都會依賴於其他 Python 庫。您可以將需求列表傳遞給 `packages_to_install`，組件將在運行時安裝這些包，然後再執行組件功能。

這類似於在 `requirements.txt` 文件中宣告對其它套件與函式庫的需求。

```python
@dsl.component(packages_to_install=['numpy==1.21.6'])
def sin(val: float = 3.14) -> float:
    return np.sin(val).item()
```

### pip_index_urls

`pip_index_urls` 公開了從默認 PyPI.org 以外的包索引中 `pip install packages_to_install` 的能力。

當您設置 `pip_index_urls` 時，KFP 將這些索引傳遞給 `pip install` 的 `--index-url` 和 `--extra-index-url` 選項。它還將每個索引設置為 `--trusted-host`。

參考以下組件範例:

```python
@dsl.component(packages_to_install=['custom-ml-package==0.0.1', 'numpy==1.21.6'],
               pip_index_urls=['http://myprivaterepo.com/simple', 'http://pypi.org/simple'],
)
def comp():
    from custom_ml_package import model_trainer
    import numpy as np
    ...
```

這些參數大致轉換為以下 pip install 命令：

```bash
pip install custom-ml-package==0.0.1 numpy==1.21.6 kfp==2 --index-url http://myprivaterepo.com/simple --trusted-host http://myprivaterepo.com/simple --extra-index-url http://pypi.org/simple --trusted-host http://pypi.org/simple
```

!!! info
    請注意，當您設置 `pip_index_urls` 時，KFP 不會自動包含 “http://pypi.org/simple”。如果您希望從私有存儲庫和默認公共存儲庫中 pip 安裝包，您應該包括私有 URL 和默認 URL，如前面的組件 `comp()` 範例所示。


## base_image

當您創建輕量級 Python 組件時，您的 Python 函數代碼將由 KFP SDK 提取，以便在管道運行時在容器內執行。默認情況下，使用的容器鏡像是 `python:3.7`。您可以通過向 `base_image` 提供參數來覆蓋此鏡像。如果您的代碼需要特定的 Python 版本或默認鏡像中未包含的其他依賴項，這將很有用。

```python
@dsl.component(base_image='python:3.8')
def print_py_version():
    import sys
    print(sys.version)
```

### install_kfp_package

`install_kfp_package` 可以與 `pip_index_urls` 一起使用，以在組件運行時提供對 kfp 包安裝的精細控制。

默認情況下，Python 組件會在運行時安裝 kfp。這是定義組件使用的符號（例如工件註釋）和訪問遠程執行組件所需的其他 KFP 庫代碼所必需的。如果 `install_kfp_package` 為 `False`，則不會通過正常的自動機制安裝 kfp。相反，您可以使用 `packages_to_install` 和 `pip_index_urls` 安裝不同版本的 kfp，可能來自非默認的 pip 索引 URL。

!!! info
    請注意，很少需要將 install_kfp_package 設置為 False，並且對於大多數用例而言，不鼓勵這樣做。

