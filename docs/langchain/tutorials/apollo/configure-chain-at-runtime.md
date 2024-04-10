# 動態修改運作中 Chain 的設定

原文: [動態修改運作中的 Chain 設定](https://myapollo.com.tw/blog/langchain-configure-chain-at-runtime/)

不知道你有沒有想過 1 個 Chain 要如何做到動態載入不同使用者的資料？或者如何像 ChatGPT 那樣可以切換模型？

這個功能在 LangChain 中稱為 `configurable_fields` 與 `configurable_alternatives` ，可以讓我們動態修改 chain 的設定，或者置換 chain 上的某個 Runnable (例如 `prompt`, `language model` 都是 Runnable) 。

能動態變更 chain 設定的功能相當重要，學會使用它也是必要的功課之一，否則我們所開發出來的應用會喪失不少彈性！

## 本文環境

- Linux (Ubuntu)
- Python 3
- LangChain

```
pip install langchain langchain-openai python-dotenv
```

## 設定 `configurable_fields` 以動態更新設定

學習如何使用 configurable_fields 之前，先看 1 個問題，以下範例讓 openai 語言模型說個笑話：

```python
from langchain_community.llms import Ollama
from langchain_core.prompts import ChatPromptTemplate


llm = Ollama(model='llama2', temperature=0)
prompt = ChatPromptTemplate.from_messages([
    ("user", "{input}"),
])
chain = prompt | llm
print(chain.invoke({'input': 'Tell me a joke'}))
```

上述範例執行之後，每次都會出現相同內容（內容如下），這是因為我們載入 llama2 時將 temperature 設定為 0 的緣故，所以語言模型的回應不會有一些隨機性產生，造成每次都回應相同內容：

```bash
Sure, here's one:

Why don't scientists trust atoms?
Because they make up everything!

I hope you found that amusing! Do you want to hear another one?
```

如果我們想動態設定一些隨機性的話，雖然可以每次產生回應都重新載入語言模型並重新設定 `temperature` 參數，但是也會導致回應時間拉長，因為語言模型的載入會需要一點點時間，也許比較好的方式是讓 chain 具有動態修改設定的能力；又或者我們想做些實驗比較不同 prompt / 參數 / 模型的差異，這時候也需要動態修改設定的能力，讓我們在呼叫 chain 時使用不同的參數，以對結果進行比較；甚至是讓 chain 具有服務多名使用者的能力，針對使用者帳號不同載入不同的語言模型、對話紀錄，就像 ChatGPT 一樣，所有使用者之間不會看到彼此的對話紀錄以確保隱私，而且付費使用者多了可以切換至更強大的語言模型的功能。

針對這些需求， LangChain 為每個 Runnable 提供 1 個稱為 `configurable_fields` 的方法，讓我們可以動態修改 chain 上的設定值。

下列是能動態設定 `temperature` 的程式碼範例：

```python
from langchain_community.llms import Ollama
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import ConfigurableField

# 設定要動態修改的設定值
llm = Ollama(model='llama2', temperature=0).configurable_fields(
    temperature=ConfigurableField(
        id="llm_temperature",
        name="LLM Temperature",
        description="The temperature of the LLM",
    )
)

# 構建一個 ChatPromptTempate 物件
prompt = ChatPromptTemplate.from_messages([
    ("user", "{input}"),
])

# 組合成一個 chain
chain = prompt | llm

# 在觸發 chain 執行之前, 設定參數
print(
    chain.with_config(
        configurable={"llm_temperature": 0.9}
    ).invoke({'input': 'Tell me a joke'})
)
```

上述範例執行結果如下，可以看到產生的內容已經產生變化：

```bash
I'm just an AI, I don't have personal preferences or emotions, but I can certainly try to come up with a joke for you! Here is one:

Why don't scientists trust atoms? Because they make up everything! 😂

I hope that brought a smile to your face! Do you have any specific topics or subjects you would like me to generate jokes about?
```

以下說明上述範例的重要部分。

為了能夠動態設定 `Ollama()` 的 `temperature` 參數，所以在 `Ollama(model='llama2', temperature=0)` 之後額外呼叫了 `configurable_fields` 方法，將 `temperature` 參數設定為 `ConfigurableField`，並將它的設定稱為 `llm_temperature` ，我們之後就可以在使用 chain 時，帶入 `llm_temperature` 的設定值， LangChain 就會知道要修改 `temperature` 參數：

```python
llm = Ollama(model='llama2', temperature=0).configurable_fields(
    temperature=ConfigurableField(
        id="llm_temperature",
        name="LLM Temperature",
        description="The temperature of the LLM",
    )
)
```

使用 chain 時，可以額外呼叫 with_config() 方法設定 temperature :

```python
chain.with_config(
    configurable={"llm_temperature": 0.9}
).invoke({'input': 'Tell me a joke'})
```

接下來，再看 1 個同時可以設定語言模型與 `temperature` 的範例：

```python
from langchain_community.llms import Ollama
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import ConfigurableField


llm = Ollama(model='llama2', temperature=0).configurable_fields(
    temperature=ConfigurableField(
        id="temperature",
        name="LLM Temperature",
        description="The temperature of the LLM",
    ),
    model=ConfigurableField(
        id="model",
        name="The Model",
        description="The language model",
    ),
)

prompt = ChatPromptTemplate.from_messages([
    ("user", "{input}"),
])
chain = prompt | llm

print(chain.with_config(
    configurable={
        "model": "codellama",
        "temperature": 0.9
    }
).invoke({'input': 'Tell me a joke'}))
```

上述範例透過 `configurable_fields()` 方法設定 2 個可動態調整的參數，分別為 `temperature` 與 `model`，並且在呼叫 chain 時設定 2 個參數， `with_config(configurable={"model": "codellama", "temperature": 0.9})`。

## 用 `configurable_alternatives` 動態置換 Runnable

先前的範例都是使用 llama2 的語言模型，這些語言模型是使用 [Ollama](https://ollama.com/) 載入，所以可以使用 `configurable_fields()` 動態設定語言模型，不過 OpenAI 的語言模型就不能使用 Ollama 載入，這使得我們無法使用 `configurable_fields()` 方法動態切換語言模型。

LangChain 還有 1 個稱為 `configurable_alternatives()` 的方法，可以動態置換 Runnable, 也就是說我們可以透過 `configurable_alternatives()` 方法，動態把 `Ollama(model='llama2')` 換成 `ChatOpenAI(model="gpt-3.5-turbo", temperature=0)`, 其範例如下：

```python
from langchain_community.llms import Ollama
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import ConfigurableField

llm = Ollama(model='llama2').configurable_alternatives(
    ConfigurableField(id="llm"),
    default_key='llama2',
    gpt35=ChatOpenAI(model="gpt-3.5-turbo", temperature=0),
)

prompt = ChatPromptTemplate.from_messages([
    ("user", "{input}"),
])

chain = prompt | llm

print(chain.with_config(configurable={"llm": "gpt35"}).invoke({'input': 'Tell me a joke'}))
```

上述範例透過 `configurable_alternatives()` 設定 1 個稱為 `llm` 的參數，該參數提供 `llama2` 與 `gpt35` 2 個選項的設定值，其中 `llama2` 是預設值，該預設值代表 `Ollama(model='llama2')` ，而 `gpt35` 代表 `ChatOpenAI(model="gpt-3.5-turbo", temperature=0) `。

所以我們可以在呼叫 chain 的時候，同樣透過 `with_config()` 方法將 `llm` 參數設定為 `gpt35` 或者 `llama2` ，也就是以下重點部分：

```python
(略).with_config(configurable={"llm": "gpt35"})
```

如此一來我們就能夠動態切換任何語言模型，或者 Runnable 。

最後再看 1 個用 `configurable_alternatives()` 動態置換 Prompt 的範例：

```python
prompt = PromptTemplate.from_template(
    "Tell me a joke about {topic}"
).configurable_alternatives(
    ConfigurableField(id="prompt"),
    default_key="joke",
    poem=PromptTemplate.from_template("Write a short poem about {topic}"),
)

chain.with_config(configurable={"prompt": "poem"}).invoke({"topic": "Earth"})
```

## 動態載入不同使用者的對話紀錄

我們在使用 ChatGPT 時， ChatGPT 會幫我們保留不同的對話紀錄，而且不會看到不屬於我們的對話紀錄，這也是做任何應用重要的功能—針對不同使用者載入專屬的設定、資料。

學會使用 `configurable_fields()` 與 `configurable_alternatives()` 之後，我們就可以動態載入使用者的相關設定與資料。

舉對話紀錄(chat history)為例， LangChain 稱此功能為 memory, 我們可以將對話紀錄存在各種 memory 中，例如檔案、資料庫等等， LangChain 提供多種整合方案可以選擇，詳細可以閱讀此[文件](https://integrations.langchain.com/memory)。

本文僅展示以 Python dictionary 儲存對話紀錄作為教學範例。

LangChain 其實有提供 1 個 Runnable 稱為 `RunnableWithMessageHistory`, 這個類別其實就是使用 `configurable_fields()` 方法，預設讓我們可以動態設定 `session_id` 藉此載入不同對話紀錄。

以下是使用 `RunnableWithMessageHistory` 的範例程式碼：

```python
from langchain_core.chat_history import BaseChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.prompts import MessagesPlaceholder
from langchain_community.chat_message_histories import ChatMessageHistory
from langchain_community.llms import Ollama


chat001 = ChatMessageHistory()

chat001.add_user_message('My name is Amo.')

store = {
    'chat001': chat001,
}


def get_chat_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in store:
        store[session_id] = ChatMessageHistory()
    return store[session_id]


llm = Ollama(model='llama2')

prompt = ChatPromptTemplate.from_messages([
    ('system', 'You are a good assistant.'),
    MessagesPlaceholder(variable_name='chat_history'),
    ('user', '{input}'),
])

chain = prompt | llm

with_message_history = RunnableWithMessageHistory(
    chain,
    get_chat_history,
    input_messages_key="input",
    history_messages_key="chat_history",
)

session_id = 'chat001'

input_text = input('>>> ')

while input_text.lower() != 'bye':
    if input_text:
        response = with_message_history.with_config(
            configurable={'session_id': session_id}
        ).invoke({'input': input_text})
        print(response)
    input_text = input('>>> ')
```

上述範例的重點部分在於使用 `RunnableWithMessageHistory()` 對 chain 進行設定，設定 1 個方法接受 `session_id` ，並回傳對話紀錄，也就是函式 `get_chat_history` , 並且設定這些對話紀錄要放進 prompt 的 `chat_history message` placeholder 中：

```python
with_message_history = RunnableWithMessageHistory(
    chain,
    get_chat_history,
    input_messages_key="input",
    history_messages_key="chat_history",
)
```

如此一來，我們就可以設定要讀取哪個 `session_id` 的對話紀錄，以新增/延續對話。

只是預設的 `get_chat_history()` 還是無法針對不同使用者進行設定，幸好 `RunnableWithMessageHistory` 還提供 `history_factory_config` 參數，可以客製化 `get_chat_history()` 所接受的參數，例如下列範例以 `history_factory_config` 參數設定 `user_id` 與 `conversation_id` 必須傳入給 `get_chat_history()` 方法，以藉此取得不同使用者的特定對話紀錄：

```python
from langchain_core.messages import HumanMessage
from langchain_core.chat_history import BaseChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.prompts import MessagesPlaceholder
from langchain_core.runnables import ConfigurableFieldSpec
from langchain_community.chat_message_histories import ChatMessageHistory
from langchain_community.llms import Ollama
from langchain_community.embeddings import OllamaEmbeddings


chat001 = ChatMessageHistory()
chat001.add_user_message('My name is Amo.')

chat002 = ChatMessageHistory()
chat002.add_user_message('My name is Jeff.')

store = {
    ('amo', 'chat001'): chat001,
    ('jeff', 'chat002'): chat002,
}


def get_chat_history(user_id: str, conversation_id: str) -> BaseChatMessageHistory:
    key = (user_id, conversation_id, )
    if key not in store:
        store[key] = ChatMessageHistory()
    return store[key]


llm = Ollama(model='llama2')

prompt = ChatPromptTemplate.from_messages([
    ('system', 'You are a good assistant.'),
    MessagesPlaceholder(variable_name='chat_history'),
    ('user', '{input}'),
])

chain = prompt | llm

with_message_history = RunnableWithMessageHistory(
    chain,
    get_chat_history,
    input_messages_key="input",
    history_messages_key="chat_history",
    history_factory_config=[
        ConfigurableFieldSpec(
            id="user_id",
            annotation=str,
            name="User Id",
            description="Unique identifier for the user.",
            default="",
            is_shared=True,
        ),
        ConfigurableFieldSpec(
            id="conversation_id",
            annotation=str,
            name="Conversation Id",
            description="Unique identifier for the conversation.",
            default="",
            is_shared=True,
        ),
    ],
)

user_id = 'jeff'
conversation_id = 'chat002'
input_text = input('>>> ')
while input_text.lower() != 'bye':
    if input_text:
        response = with_message_history.with_config(
            configurable={
                'user_id': user_id,
                'conversation_id': conversation_id
            }
        ).invoke({'input': input_text})
        print(response)
    input_text = input('>>> ')
```

上述範例的重點在於我們在 `history_factory_config` 參數中以 `ConfigurableFieldSpec()` 指定 2 個設定，分別為 `user_id` 與 `conversation_id` ，這 2 個設定都是字串型態：

```python
    history_factory_config=[
        ConfigurableFieldSpec(
            id="user_id",
            annotation=str,
            name="User Id",
            description="Unique identifier for the user.",
            default="",
            is_shared=True,
        ),
        ConfigurableFieldSpec(
            id="conversation_id",
            annotation=str,
            name="Conversation Id",
            description="Unique identifier for the conversation.",
            default="",
            is_shared=True,
        ),
    ],
```

如此就可以在呼叫 chain 時，帶入 `user_id` 與 `conversation_id` 2 個設定，做到動態載入使用者資料的功能（當然 `get_chat_history` 函式也需要做相對應的邏輯修改）：

```python
with_message_history.with_config(
    configurable={
        'user_id': user_id,
        'conversation_id': conversation_id
    }
).invoke({'input': input_text})
```

至此，你已經理解如何做到 1 個 chain 服務多個使用者的基本架構。

## 總結

動態修改 chain 的設定是 LangChain 重要的一項技術，學會此技術才能夠做到動態載入使用者設定、資料等功能。