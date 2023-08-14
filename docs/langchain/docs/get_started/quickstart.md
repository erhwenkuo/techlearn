# 快速開始

## 安裝

要安裝 LangChain 運行下列命令:

=== "pip"

    ```bash
    pip install langchain
    ```

=== "Conda"

    ```bash
    conda install langchain -c conda-forge
    ```

有關更多詳細信息，請參閱我們的[安裝指南](./installation.md)。

## 環境設置

使用 LangChain 通常需要與一個或多個模型 (llm) 提供者、數據存儲、API 等集成。在本例中，我們將使用 OpenAI 的模型 API。

首先我們需要安裝他們的 Python 套件:

```bash
pip install openai
```

訪問 API 需要 API 密鑰，您可以通過創建帳戶並前往[此處](https://platform.openai.com/account/api-keys)獲取該密鑰。一旦我們有了密鑰，我們就需要通過運行以下命令將其設置為環境變量：

```bash
export OPENAI_API_KEY="..."
```

如果您不想設置環境變量，也可以在啟動 OpenAI LLM 類別時直接通過 `openai_api_key` 命名參數傳遞密鑰:

```python
from langchain.llms import OpenAI

llm = OpenAI(openai_api_key="...")
```

## 構建應用程式

現在我們可以開始構建我們的語言模型應用程式。 LangChain 提供了許多可用於構建語言模型應用程式的模塊。模塊可以在簡單的應用程式中獨立使用，也可以組合起來用於更複雜的用例。

LangChain 應用程式的核心構建模塊是 **LLMChain**。它結合了三件事:

- `LLM`: 語言模型是這裡的核心推理引擎。為了使用 LangChain，您需要了解不同類型的語言模型以及如何了解如何使用它們。
- `Prompt Templates`: 為語言模型提供了指令。這控制著語言模型的輸出內容，因此了解如何構建提示和不同的提示策略至關重要。
- `Output Parsers`: 這將 LLM 的原始回應/輸出轉換為更容易理解的格式，從而可以被下游應用輕鬆來使用。

在本入門指南中，我們將單獨介紹這三個組件，然後介紹結合了所有組件的 LLMChain。了解這些概念將為您使用和定制 LangChain 應用程序做好準備。大多數 LangChain 應用程序允許您配置 LLM 和/或使用的提示，因此了解如何利用這一點將是一個很大的推動因素。

### LLM

語言模型有兩種類型，在 LangChain 中稱為:

- **LLM**: 這是一個語言模型，它接受一個 string 作為輸入並返回一個 string。

- **ChatModel**: 這是一個語言模型，它將 message 列表作為輸入並返回 message。

LLM 的輸入/輸出簡單易懂: `一個 string`。但是 ChatModels 呢？ 

輸入是 `ChatMessage` 列表，輸出是單個 `ChatMessage`。 `ChatMessage` 有兩個必需的組件：

- `content`: 這是消息的內容。
- `role`: 這是 `ChatMessage` 來源實體的角色。

LangChain 提供了幾個物件來方便區分不同的角色：

- `HumanMessage`: 來自人類/用戶的 `ChatMessage`。
- `AIMessage`: 來自人工智能/助手的 `ChatMessage`。
- `SystemMessage`: 來自系統的 `ChatMessage`。
- `FunctionMessage`: 來自函數調用的 `ChatMessage`。

如果這些角色聽起來都不合適，還有一個 `ChatMessage` 類別，您可以在其中手動指定角色。有關如何最有效地使用這些不同消息的更多信息，請參閱我們的提示指南。

LangChain 為兩者提供了標準接口，但了解這種差異對於為給定語言模型構建提示很有用。 LangChain 暴露的標準接口有兩個方法：

- `predict`: 接收一個 string，返回一個 string。
- `predict_messages`: 接收 message 列表，返回一個 message。

讓我們看看如何使用這些不同類型的模型和這些不同類型的輸入。首先，我們導入一個 LLM 和一個 ChatModel。

```python
from langchain.llms import OpenAI
from langchain.chat_models import ChatOpenAI

llm = OpenAI()
chat_model = ChatOpenAI()

llm.predict("hi!")
# >> "Hi"

chat_model.predict("hi!")
# >> "Hi"
```

`OpenAI` 和 `ChatOpenAI` 物件基本上只是配置物件。您可以使用 `temperature` 等參數初始化它們，然後傳遞它們。

接下來，讓我們使用 `predict` 方法來運行字符串輸入。

```python
text = "What would be a good company name for a company that makes colorful socks?"

llm.predict(text)
# >> Feetful of Fun

chat_model.predict(text)
# >> Socks O'Color
```

最後，讓我們使用 `Predict_messages` 方法來運行消息列表。

```python
from langchain.schema import HumanMessage

text = "What would be a good company name for a company that makes colorful socks?"
messages = [HumanMessage(content=text)]

llm.predict_messages(messages)
# >> Feetful of Fun

chat_model.predict_messages(messages)
# >> Socks O'Color
```

對於這兩種方法，您還可以傳入參數作為關鍵字參數。例如，您可以傳入 `temperature=0` 來調整物件配置的 `temperature`。在 runtime 傳入的任何值都將始終覆蓋物件初始配置的值。

### Prompt template

大多數 LLM 應用不會將用戶輸入直接傳遞到 LLM。通常，他們會將用戶輸入添加到較大的文本中，稱為 `prompt template`，該文本提供有關當前特定任務的附加上下文。

在前面的示例中，我們傳遞給模型的文本包含生成公司名稱的指令。對於我們的應用程序，如果用戶只需提供公司/產品的描述，而不必擔心提供模型說明，那就太好了。

`PromptTemplate` 正好可以幫助解決這個問題！它們綁定了從用戶輸入到格式化的提示的所有邏輯。這可以非常簡單地開始 - 例如，生成上述字符串的提示就是:

```python
from langchain.prompts import PromptTemplate

prompt = PromptTemplate.from_template("What is a good name for a company that makes {product}?")

prompt.format(product="colorful socks")
```

結果:

```bash
What is a good name for a company that makes colorful socks?
```

然而，與原始字符串格式化相比，使用它們有幾個優點。您可以“部分”輸出變量 - 例如，您一次只能格式化部分變量。您可以將它們組合在一起，輕鬆地將不同的模板組合成一個提示。有關這些功能的說明，請參閱 [section on prompts ](https://python.langchain.com/docs/modules/model_io/prompts)以了解更多詳細信息。

`PromptTemplate` 還可用於生成消息列表。在這種情況下，提示不僅包含有關內容的信息，還包含每條消息（其角色、在列表中的位置等）。這裡，最常發生的是 `ChatPromptTemplate` 是 `ChatMessageTemplate` 的列表。每個 `ChatMessageTemplate` 都包含有關如何格式化該 `ChatMessage` 的說明 - 它的角色，以及它的內容。讓我們看看下面這個:

```python
from langchain.prompts.chat import (
    ChatPromptTemplate,
    SystemMessagePromptTemplate,
    HumanMessagePromptTemplate,
)

template = "You are a helpful assistant that translates {input_language} to {output_language}."

system_message_prompt = SystemMessagePromptTemplate.from_template(template)

human_template = "{text}"

human_message_prompt = HumanMessagePromptTemplate.from_template(human_template)

chat_prompt = ChatPromptTemplate.from_messages([system_message_prompt, human_message_prompt])

chat_prompt.format_messages(input_language="English", output_language="French", text="I love programming.")
```

結果:

```bash
[
    SystemMessage(content="You are a helpful assistant that translates English to French.", additional_kwargs={}),
    HumanMessage(content="I love programming.")
]
```

`ChatPromptTemplate` 還可以包含除 `ChatMessageTemplate` 之外的其他內容 - 請參閱 [section on prompts ](https://python.langchain.com/docs/modules/model_io/prompts) 以獲取更多詳細信息。

### Output Parser

`OutputParser` 將 LLM 的原始輸出轉換為可在下游使用的格式。 `OutputParser` 有幾種主要類型，包括：

- 從 LLM 轉換文本 -> 結構化信息（例如 JSON）
- 將 ChatMessage 轉換為字符串
- 將除消息之外的調用返回的額外信息（如 OpenAI 函數調用）轉換為字符串。

有關這方面的完整信息，請參閱 [section on output parsers](https://python.langchain.com/docs/modules/model_io/output_parsers)

在本入門指南中，我們將編寫自己的輸出解析器 - 將逗號分隔列表轉換為列表的解析器。

```python
from langchain.schema import BaseOutputParser

class CommaSeparatedListOutputParser(BaseOutputParser):
    """Parse the output of an LLM call to a comma-separated list."""
    def parse(self, text: str):
        """Parse the output of an LLM call."""
        return text.strip().split(", ")

CommaSeparatedListOutputParser().parse("hi, bye")
# >> ['hi', 'bye']
```

### LLMChain

我們現在可以將所有這些組合成一條鏈。該鏈將獲取輸入變量，將這些變量傳遞給提示模板以創建提示，將提示傳遞給 LLM，然後通過（可選）輸出解析器傳遞輸出。這是捆綁模塊化邏輯塊的便捷方法。讓我們看看它的實際效果！

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts.chat import (
    ChatPromptTemplate,
    SystemMessagePromptTemplate,
    HumanMessagePromptTemplate,
)
from langchain.chains import LLMChain
from langchain.schema import BaseOutputParser

class CommaSeparatedListOutputParser(BaseOutputParser):
    """Parse the output of an LLM call to a comma-separated list."""

    def parse(self, text: str):
        """Parse the output of an LLM call."""
        return text.strip().split(", ")

template = """You are a helpful assistant who generates comma separated lists.
A user will pass in a category, and you should generate 5 objects in that category in a comma separated list.
ONLY return a comma separated list, and nothing more."""
system_message_prompt = SystemMessagePromptTemplate.from_template(template)
human_template = "{text}"
human_message_prompt = HumanMessagePromptTemplate.from_template(human_template)

chat_prompt = ChatPromptTemplate.from_messages([system_message_prompt, human_message_prompt])
chain = LLMChain(
    llm=ChatOpenAI(),
    prompt=chat_prompt,
    output_parser=CommaSeparatedListOutputParser()
)
chain.run("colors")
# >> ['red', 'blue', 'green', 'yellow', 'orange']
```

## 下一步

我們現在已經了解瞭如何創建 LangChain 應用程序的核心構建模組 `LLMChain`。所有這些組件 (`LLM`, `prompt`, `output parser`) 都有更多的細緻的功能，還有更多不同的組件需要學習。

讓我們繼續 LangChain 的學習旅程：

- [深入研究 LLM](https://python.langchain.com/docs/modules/model_io)、提示和輸出解析器
- 了解其他[關鍵組件](https://python.langchain.com/docs/modules)
- 查看我們的[有用指南](https://python.langchain.com/docs/guides)，了解特定主題的詳細演練
- 探索[端到端用例](https://python.langchain.com/docs/use_cases)