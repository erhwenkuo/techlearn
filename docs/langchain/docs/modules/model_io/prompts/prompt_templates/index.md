# Prompt templates

提示模板是用於生成語言模型提示的預定義配方。

模板可能包括指示、few shot examples,以及適合給定任務的特定上下文和問題。

LangChain 提供了創建和使用提示模板的工具。

LangChain 致力於創建與模型無關的模板，以便能夠輕鬆地跨不同語言模型重用現有模板。

通常，語言模型期望提示是 string 或 messages 列表。

## Prompt template

使用 `PromptTemplate` 為 string 提示創建模板。

默認情況下，`PromptTemplate` 使用 Python 的 `str.format` 語法進行模板化；但是也可以使用其他模板語法（例如 `jinja2`）。

```python
from langchain import PromptTemplate

prompt_template = PromptTemplate.from_template(
    "Tell me a {adjective} joke about {content}."
)

prompt_template.format(adjective="funny", content="chickens")
```

結果:

```bash
"Tell me a funny joke about chickens."
```

該模板支持任意數量的變數，包括無變數：

```python
from langchain import PromptTemplate

prompt_template = PromptTemplate.from_template(
    "Tell me a joke"
)

prompt_template.format()
```

對於附加驗證，請顯式指定 `input_variables`。這些變數將在實例化期間與模板字符串中存在的變量進行比較，如果不匹配則引發異常；例如，

```python
from langchain import PromptTemplate

invalid_prompt = PromptTemplate(
    input_variables=["adjective"],
    template="Tell me a {adjective} joke about {content}."
)
```

您可以創建自定義提示模板，以您想要的任何方式格式化提示。有關更多信息，請參閱[自定義提示模板](https://python.langchain.com/docs/modules/model_io/prompts/prompt_templates/custom_prompt_template.html)。

## Chat prompt template

聊天模型的提示是聊天消息列表。

每條聊天消息都與內容以及稱為角色的附加參數相關聯。例如，在 OpenAI Chat Completions API 中，聊天消息可以與 AI 助手、人類或系統角色相關聯。

創建一個這樣的聊天提示模板：

```python
from langchain.prompts import ChatPromptTemplate

template = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful AI bot. Your name is {name}."),
    ("human", "Hello, how are you doing?"),
    ("ai", "I'm doing well, thanks!"),
    ("human", "{user_input}"),
])

messages = template.format_messages(
    name="Bob",
    user_input="What is your name?"
)
```

`ChatPromptTemplate.from_messages` 接受各種消息表示形式。

例如，除了使用上面使用的 `(type, content)` 的 2-tuple 表示之外，您還可以傳入 `MessagePromptTemplate` 或 `BaseMessage` 的實例。

```python
from langchain.prompts import ChatPromptTemplate
from langchain.prompts.chat import SystemMessage, HumanMessagePromptTemplate

template = ChatPromptTemplate.from_messages(
    [
        SystemMessage(
            content=(
                "You are a helpful assistant that re-writes the user's text to "
                "sound more upbeat."
            )
        ),
        HumanMessagePromptTemplate.from_template("{text}"),
    ]
)

from langchain.chat_models import ChatOpenAI

llm = ChatOpenAI()
llm(template.format_messages(text='i dont like eating tasty things.'))
```

結果:

```bash
AIMessage(content='I absolutely adore indulging in delicious treats!', additional_kwargs={}, example=False)
```

這為您構建聊天提示的方式提供了很大的靈活性。





