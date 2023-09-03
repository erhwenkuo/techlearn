# 連接到 Feature Store

Featrue store 是傳統機器學習的一個概念，可確保輸入模型的數據是最新的且相關的。有關這方面的更多信息，請參見[此處](https://www.tecton.ai/blog/what-is-a-feature-store/)。

當考慮將 LLM 應用程序投入生產時，這個概念非常重要。為了客製化 LLM 應用，您可能希望將 LLM 與有關特定用戶的最新信息結合起來。Feature store 是保持數據新鮮的好方法，LangChain 提供了一種將數據與 LLM 結合起來的簡單方法。

在此筆記本中，我們將展示如何將提示模板連接到特徵存儲。基本思想是從提示模板內部調用特徵存儲來檢索然後格式化為提示的值。

## Feast

首先，我們將使用流行的開源特徵存儲框架 [Feast](https://github.com/feast-dev/feast)。

這假設您已經有關 LangChain 入門的相關設定步驟。我們將在以 quickstart 示例為基礎，並創建 `LLMChain` 來向特定驅動程序寫入有關其最新統計數據的註釋。

### 載入 Feast Store

同樣，這應該根據 [Feast README](https://github.com/feast-dev/feast/blob/master/README.md) 中的說明進行設置

```python
from feast import FeatureStore

# You may need to update the path depending on where you stored it
feast_repo_path = "../../../../../my_feature_repo/feature_repo/"
store = FeatureStore(repo_path=feast_repo_path)
```

### Prompts

在這裡我們將設置一個自定義的 `FeastPromptTemplate`。此提示模板將接收 `driver id`、查找其統計數據並將這些統計數據格式化為提示(prompt)。

請注意，此提示模板的輸入只是 `driver_id`，因為這是唯一的用戶定義部分（所有其他變量都在提示模板內查找）。

```python
from langchain.prompts import PromptTemplate, StringPromptTemplate
```

**API Reference:**

- `PromptTemplate` 從 `langchain.prompts`
- `StringPromptTemplate` 從 `langchain.prompts`

```python
template = """Given the driver's up to date stats, write them note relaying those stats to them.
If they have a conversation rate above .5, give them a compliment. Otherwise, make a silly joke about chickens at the end to make them feel better

Here are the drivers stats:
Conversation rate: {conv_rate}
Acceptance rate: {acc_rate}
Average Daily Trips: {avg_daily_trips}

Your response:"""

prompt = PromptTemplate.from_template(template)
```

```python
class FeastPromptTemplate(StringPromptTemplate):
    def format(self, **kwargs) -> str:
        driver_id = kwargs.pop("driver_id")
        feature_vector = store.get_online_features(
            features=[
                "driver_hourly_stats:conv_rate",
                "driver_hourly_stats:acc_rate",
                "driver_hourly_stats:avg_daily_trips",
            ],
            entity_rows=[{"driver_id": driver_id}],
        ).to_dict()
        kwargs["conv_rate"] = feature_vector["conv_rate"][0]
        kwargs["acc_rate"] = feature_vector["acc_rate"][0]
        kwargs["avg_daily_trips"] = feature_vector["avg_daily_trips"][0]
        return prompt.format(**kwargs)
```

```python
prompt_template = FeastPromptTemplate(input_variables=["driver_id"])
```

```python
print(prompt_template.format(driver_id=1001))
```

結果:

```bash
Given the driver's up to date stats, write them note relaying those stats to them.
If they have a conversation rate above .5, give them a compliment. Otherwise, make a silly joke about chickens at the end to make them feel better

Here are the drivers stats:
Conversation rate: 0.4745151400566101
Acceptance rate: 0.055561766028404236
Average Daily Trips: 936

Your response:
```

### 在 chain 中使用

我們現在可以在鏈中使用它，成功創建一個由特徵存儲支持實現個性化的鏈

```python
from langchain.chat_models import ChatOpenAI
from langchain.chains import LLMChain
```

**API Reference:**

- [ChatOpenAI](https://api.python.langchain.com/en/latest/chat_models/langchain.chat_models.openai.ChatOpenAI.html) 從 `langchain.chat_models`
- [LLMChain](https://api.python.langchain.com/en/latest/chains/langchain.chains.llm.LLMChain.html) 從 `langchain.chains`

```python
chain = LLMChain(llm=ChatOpenAI(), prompt=prompt_template)
```

```python
chain.run(1001)
```

結果:

```bash
"Hi there! I wanted to update you on your current stats. Your acceptance rate is 0.055561766028404236 and your average daily trips are 936. While your conversation rate is currently 0.4745151400566101, I have no doubt that with a little extra effort, you'll be able to exceed that .5 mark! Keep up the great work! And remember, even chickens can't always cross the road, but they still give it their best shot."
```
