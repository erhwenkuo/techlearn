# Agent Types

這按照幾個維度對所有可用代理進行分類。

**Intended Model Type**

該代理是否適用於聊天模型（接收訊息，輸出訊息）或 LLM（接收字串，輸出字串）。這影響的主要因素是所使用的提示策略。您可以使用具有與預期不同類型模型的代理，但它可能不會產生相同品質的結果。

**Supports Chat History**

這些代理類型是否支援聊天歷史記錄。如果是，則表示它可以用作聊天機器人。如果沒有，那就意味著它更適合單一任務。支援聊天歷史通常需要更好的模型，因此針對較差模型的早期代理類型可能不支援它。

**Supports Multi-Input Tools**

這些代理類型是否支援具有多個輸入的工具。如果一個工具只需要一個輸入，法學碩士通常更容易知道如何調用它。因此，針對較差模型的幾種早期代理類型可能不支援它們。

**Supports Parallel Function Calling**

讓 LLM 同時調用多個工具可以大大加快代理的速度，無論是否有任務需要這樣做。然而，對於 LLM 來說，做到這一點更具挑戰性，因此某些代理類型不支持這一點。

**Required Model Params**

該代理是否需要模型支援任何附加參數。某些代理類型利用 OpenAI 函數呼叫等功能，這需要其他模型參數。如果不需要，則表示一切都是透過提示完成的

**When to Use**

我們對何時應考慮使用此代理類型的評論。

|Agent Type	|Intended Model Type	|Supports Chat History	|Supports Multi-Input Tools	|Supports Parallel Function Calling	|Required Model Params	|When to Use	|API|
|---|---|---|---|---|---|---|---|
|OpenAI Tools	|Chat	|✅	|✅	|✅	|tools	|If you are using a recent OpenAI model (1106 onwards)	|Ref|
|OpenAI Functions	|Chat	|✅	|✅	|	|functions	|If you are using an OpenAI model, or an open-source model that has been finetuned for function calling and exposes the same functions parameters as OpenAI	|Ref|
|XML	|LLM	|✅	|	|	|	|If you are using Anthropic models, or other models good at XML	|Ref|
|Structured Chat	|Chat	|✅	|✅	|	|	|If you need to support tools with multiple inputs	|Ref|
|JSON Chat	|Chat	|✅	|	|	|	|If you are using a model good at JSON	|Ref|
|ReAct	|LLM	|✅	|	|	|	|If you are using a simple model	|Ref|
|Self Ask With Search	|LLM	|	|	|	|	|If you are using a simple model and only have one search tool	|Ref|

