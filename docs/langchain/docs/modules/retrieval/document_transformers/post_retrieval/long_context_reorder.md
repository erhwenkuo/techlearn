# Lost in the middle: The problem with long contexts

無論模型的架構為何，當您包含 10 多個檢索到的文件時，效能都會大幅下降。簡而言之：當模型必須在長上下文中存取相關資訊時，它們往往會忽略所提供的文件。參見：https://arxiv.org/abs/2307.03172

為了避免此問題，您可以在檢索後對文件重新排序，以避免效能下降。

```python
import os
import chromadb
from langchain.vectorstores import Chroma
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.document_transformers import (
    LongContextReorder,
)
from langchain.chains import StuffDocumentsChain, LLMChain
from langchain.prompts import PromptTemplate
from langchain.llms import OpenAI

# Get embeddings.
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")

texts = [
    "Basquetball is a great sport.",
    "Fly me to the moon is one of my favourite songs.",
    "The Celtics are my favourite team.",
    "This is a document about the Boston Celtics",
    "I simply love going to the movies",
    "The Boston Celtics won the game by 20 points",
    "This is just a random text.",
    "Elden Ring is one of the best games in the last 15 years.",
    "L. Kornet is one of the best Celtics players.",
    "Larry Bird was an iconic NBA player.",
]

# Create a retriever
retriever = Chroma.from_texts(texts, embedding=embeddings).as_retriever(
    search_kwargs={"k": 10}
)
query = "What can you tell me about the Celtics?"

# Get relevant documents ordered by relevance score
docs = retriever.get_relevant_documents(query)

print(docs)
```

**API Reference:**

- [Chroma](https://api.python.langchain.com/en/latest/vectorstores/langchain.vectorstores.chroma.Chroma.html)
- [HuggingFaceEmbeddings](https://api.python.langchain.com/en/latest/embeddings/langchain.embeddings.huggingface.HuggingFaceEmbeddings.html)
- [LongContextReorder](https://api.python.langchain.com/en/latest/document_transformers/langchain.document_transformers.long_context_reorder.LongContextReorder.html)
- [StuffDocumentsChain](https://api.python.langchain.com/en/latest/chains/langchain.chains.combine_documents.stuff.StuffDocumentsChain.html)
- [LLMChain](https://api.python.langchain.com/en/latest/chains/langchain.chains.llm.LLMChain.html)
- [PromptTemplate](https://api.python.langchain.com/en/latest/prompts/langchain.prompts.prompt.PromptTemplate.html)
- [OpenAI](https://api.python.langchain.com/en/latest/llms/langchain.llms.openai.OpenAI.html)

```python
[Document(page_content='This is a document about the Boston Celtics', metadata={}),
    Document(page_content='The Celtics are my favourite team.', metadata={}),
    Document(page_content='L. Kornet is one of the best Celtics players.', metadata={}),
    Document(page_content='The Boston Celtics won the game by 20 points', metadata={}),
    Document(page_content='Larry Bird was an iconic NBA player.', metadata={}),
    Document(page_content='Elden Ring is one of the best games in the last 15 years.', metadata={}),
    Document(page_content='Basquetball is a great sport.', metadata={}),
    Document(page_content='I simply love going to the movies', metadata={}),
    Document(page_content='Fly me to the moon is one of my favourite songs.', metadata={}),
    Document(page_content='This is just a random text.', metadata={})]
```

```python
# Reorder the documents:
# Less relevant document will be at the middle of the list and more
# relevant elements at beginning / end.
reordering = LongContextReorder()
reordered_docs = reordering.transform_documents(docs)

# Confirm that the 4 relevant documents are at beginning and end.
reordered_docs
```

```python
    [Document(page_content='The Celtics are my favourite team.', metadata={}),
     Document(page_content='The Boston Celtics won the game by 20 points', metadata={}),
     Document(page_content='Elden Ring is one of the best games in the last 15 years.', metadata={}),
     Document(page_content='I simply love going to the movies', metadata={}),
     Document(page_content='This is just a random text.', metadata={}),
     Document(page_content='Fly me to the moon is one of my favourite songs.', metadata={}),
     Document(page_content='Basquetball is a great sport.', metadata={}),
     Document(page_content='Larry Bird was an iconic NBA player.', metadata={}),
     Document(page_content='L. Kornet is one of the best Celtics players.', metadata={}),
     Document(page_content='This is a document about the Boston Celtics', metadata={})]
```

```python
# We prepare and run a custom Stuff chain with reordered docs as context.

# Override prompts
document_prompt = PromptTemplate(
    input_variables=["page_content"], template="{page_content}"
)
document_variable_name = "context"
llm = OpenAI()
stuff_prompt_override = """Given this text extracts:
-----
{context}
-----
Please answer the following question:
{query}"""
prompt = PromptTemplate(
    template=stuff_prompt_override, input_variables=["context", "query"]
)

# Instantiate the chain
llm_chain = LLMChain(llm=llm, prompt=prompt)
chain = StuffDocumentsChain(
    llm_chain=llm_chain,
    document_prompt=document_prompt,
    document_variable_name=document_variable_name,
)
chain.run(input_documents=reordered_docs, query=query)
```



