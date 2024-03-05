# Toolkits

工具包是旨在一起用於特定任務並具有方便的載入方法的工具的集合。有關這些的完整列表，請訪問[Integrations](https://python.langchain.com/docs/integrations/toolkits/)。

所有工具包都公開一個 `get_tools` 方法，該方法傳回工具清單。因此，您可以這樣做：

```python
# Initialize a toolkit
toolkit = ExampleTookit(...)

# Get list of tools
tools = toolkit.get_tools()

# Create agent
agent = create_agent_method(llm, tools, prompt)
```
