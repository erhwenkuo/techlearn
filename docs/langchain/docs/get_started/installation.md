# 安裝

## 正式發布版本

要安裝 LangChain 運行下列命令:

=== "pip"

    ```bash
    pip install langchain
    ```

=== "Conda"

    ```bash
    conda install langchain -c conda-forge
    ```

這將安裝 LangChain 的最低要求。 LangChain 的很多價值在於將其與各種模型提供程序、數據存儲等集成。默認情況下，未安裝執行此操作所需的依賴項。然而，還有另外兩種安裝 LangChain 的方法可以引入這些依賴項。

要安裝常見 LLM 提供商所需的模塊，請運行：

```bash
pip install langchain[llms]
```

要安裝所有集成所需的所有模組，請運行：

```bash
pip install langchain[all]
```

## 從源碼

如果您想從源碼安裝，可以通過 clone 存儲庫並運行：

```bash
pip install -e .
```