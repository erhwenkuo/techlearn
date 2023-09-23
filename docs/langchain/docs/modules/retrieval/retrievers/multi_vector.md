# MultiVector Retriever

每個文件儲存多個向量通常是有益的。在許多用例中，這是有益的。 LangChain 有一個基礎 `MultiVectorRetriever`，可以輕鬆查詢此類設定。很多複雜性在於如何為每個文件建立多個向量。本文涵蓋了創建這些向量和使用 `MultiVectorRetriever` 的一些常見方法。

為每個文件建立多個向量的方法包括：

- `Smaller chunks`: 將文件分割成較小的區塊，然後嵌入這些區塊（這是 ParentDocumentRetriever）。
- `Summary`: 為每個文件建立摘要，將其與文件一起嵌入（或取代文件）。
- `Hypothetical questions`: 建立每個文件都適合回答的假設性問題，將這些問題與文件一起嵌入（或取代文件）。

請注意，這也啟用了另一種添加嵌入的方法 - manually。這很棒，因為您可以明確添加導致文件恢復的問題或查詢，從而為您提供更多控制權。

```python
from langchain.retrievers.multi_vector import MultiVectorRetriever
```

**API Reference:**

- [MultiVectorRetriever](https://api.python.langchain.com/en/latest/retrievers/langchain.retrievers.multi_vector.MultiVectorRetriever.html)

```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.storage import InMemoryStore
from langchain.document_loaders import TextLoader
```

**API Reference:**

- [Chroma](https://api.python.langchain.com/en/latest/vectorstores/langchain.vectorstores.chroma.Chroma.html)
- [OpenAIEmbeddings](https://api.python.langchain.com/en/latest/embeddings/langchain.embeddings.openai.OpenAIEmbeddings.html)
- [RecursiveCharacterTextSplitter](https://api.python.langchain.com/en/latest/text_splitter/langchain.text_splitter.RecursiveCharacterTextSplitter.html)
- [InMemoryStore](https://api.python.langchain.com/en/latest/storage/langchain.storage.in_memory.InMemoryStore.html)
- [TextLoader](https://api.python.langchain.com/en/latest/document_loaders/langchain.document_loaders.text.TextLoader.html)


```python
loaders = [
    TextLoader('../../paul_graham_essay.txt'),
    TextLoader('../../state_of_the_union.txt'),
]

docs = []

for l in loaders:
    docs.extend(l.load())

text_splitter = RecursiveCharacterTextSplitter(chunk_size=10000)

docs = text_splitter.split_documents(docs)
```

## Smaller chunks

---

通常，檢索較大的資訊區塊但嵌入較小的資訊區塊可能很有用。這允許嵌入盡可能接近地捕獲語義，但可以將盡可能多的上下文傳遞到下游。請注意，這就是 `ParentDocumentRetriever` 的作用。在這裡，我們展示了幕後發生的事情。

```python
# The vectorstore to use to index the child chunks
vectorstore = Chroma(
    collection_name="full_documents",
    embedding_function=OpenAIEmbeddings()
)

# The storage layer for the parent documents
store = InMemoryStore()
id_key = "doc_id"

# The retriever (empty to start)
retriever = MultiVectorRetriever(
    vectorstore=vectorstore, 
    docstore=store, 
    id_key=id_key,
)

import uuid

doc_ids = [str(uuid.uuid4()) for _ in docs]

# The splitter to use to create smaller chunks
child_text_splitter = RecursiveCharacterTextSplitter(chunk_size=400)

sub_docs = []

for i, doc in enumerate(docs):
    _id = doc_ids[i]
    _sub_docs = child_text_splitter.split_documents([doc])

    for _doc in _sub_docs:
        _doc.metadata[id_key] = _id
    sub_docs.extend(_sub_docs)

retriever.vectorstore.add_documents(sub_docs)
retriever.docstore.mset(list(zip(doc_ids, docs)))

# Vectorstore alone retrieves the small chunks
retriever.vectorstore.similarity_search("justice breyer")[0]
```

結果:

```
    Document(page_content='Tonight, I’d like to honor someone who has dedicated his life to serve this country: Justice Stephen Breyer—an Army veteran, Constitutional scholar, and retiring Justice of the United States Supreme Court. Justice Breyer, thank you for your service. \n\nOne of the most serious constitutional responsibilities a President has is nominating someone to serve on the United States Supreme Court.', metadata={'doc_id': '10e9cbc0-4ba5-4d79-a09b-c033d1ba7b01', 'source': '../../state_of_the_union.txt'})
```

```python
# Retriever returns larger chunks
len(retriever.get_relevant_documents("justice breyer")[0].page_content)
```

結果:

```
    9874
```

## Summary

---

通常，摘要可以更準確地提煉出某個區塊的內容，從而實現更好的檢索。在這裡，我們展示如何建立摘要，然後嵌入它們。

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.schema.output_parser import StrOutputParser
import uuid
from langchain.schema.document import Document
```

**API Reference:**

- [ChatOpenAI](https://api.python.langchain.com/en/latest/chat_models/langchain.chat_models.openai.ChatOpenAI.html)
- [ChatPromptTemplate](https://api.python.langchain.com/en/latest/prompts/langchain.prompts.chat.ChatPromptTemplate.html)
- [StrOutputParser](https://api.python.langchain.com/en/latest/schema/langchain.schema.output_parser.StrOutputParser.html)
- [Document](https://api.python.langchain.com/en/latest/schema/langchain.schema.document.Document.html)

```python
chain = (
    {"doc": lambda x: x.page_content}
    | ChatPromptTemplate.from_template("Summarize the following document:\n\n{doc}")
    | ChatOpenAI(max_retries=0)
    | StrOutputParser()
)

summaries = chain.batch(docs, {"max_concurrency": 5})

# The vectorstore to use to index the child chunks
vectorstore = Chroma(
    collection_name="summaries",
    embedding_function=OpenAIEmbeddings()
)
# The storage layer for the parent documents
store = InMemoryStore()
id_key = "doc_id"
# The retriever (empty to start)
retriever = MultiVectorRetriever(
    vectorstore=vectorstore, 
    docstore=store, 
    id_key=id_key,
)

doc_ids = [str(uuid.uuid4()) for _ in docs]

summary_docs = [Document(page_content=s,metadata={id_key: doc_ids[i]}) for i, s in enumerate(summaries)]

retriever.vectorstore.add_documents(summary_docs)
retriever.docstore.mset(list(zip(doc_ids, docs)))

# # We can also add the original chunks to the vectorstore if we so want
# for i, doc in enumerate(docs):
#     doc.metadata[id_key] = doc_ids[i]
# retriever.vectorstore.add_documents(docs)

sub_docs = vectorstore.similarity_search("justice breyer")

print(sub_docs[0])
```

結果:

```
    Document(page_content="The document is a transcript of a speech given by the President of the United States. The President discusses several important issues and initiatives, including the nomination of a Supreme Court Justice, border security and immigration reform, protecting women's rights, advancing LGBTQ+ equality, bipartisan legislation, addressing the opioid epidemic and mental health, supporting veterans, investigating the health effects of burn pits on military personnel, ending cancer, and the strength and resilience of the American people.", metadata={'doc_id': '79fa2e9f-28d9-4372-8af3-2caf4f1de312'})
```

```python
retrieved_docs = retriever.get_relevant_documents("justice breyer")

len(retrieved_docs[0].page_content)
```

結果:

```
    9194
```

## Hypothetical Queries

---

LLM 也可以用於產生針對特定文件可能提出的假設問題清單。然後可以嵌入這些問題

```python
functions = [
    {
      "name": "hypothetical_questions",
      "description": "Generate hypothetical questions",
      "parameters": {
        "type": "object",
        "properties": {
          "questions": {
            "type": "array",
            "items": {
                "type": "string"
              },
          },
        },
        "required": ["questions"]
      }
    }
  ]

from langchain.output_parsers.openai_functions import JsonKeyOutputFunctionsParser

chain = (
    {"doc": lambda x: x.page_content}
    # Only asking for 3 hypothetical questions, but this could be adjusted
    | ChatPromptTemplate.from_template("Generate a list of 3 hypothetical questions that the below document could be used to answer:\n\n{doc}")
    | ChatOpenAI(max_retries=0, model="gpt-4").bind(functions=functions, function_call={"name": "hypothetical_questions"})
    | JsonKeyOutputFunctionsParser(key_name="questions")
)
```

**API Reference:**

- [JsonKeyOutputFunctionsParser](https://api.python.langchain.com/en/latest/output_parsers/langchain.output_parsers.openai_functions.JsonKeyOutputFunctionsParser.html)

```python
chain.invoke(docs[0])
```

結果:

```
    ["What was the author's initial impression of philosophy as a field of study, and how did it change when they got to college?",
     'Why did the author decide to switch their focus to Artificial Intelligence (AI)?',
     "What led to the author's disillusionment with the field of AI as it was practiced at the time?"]
```

```python
hypothetical_questions = chain.batch(docs, {"max_concurrency": 5})

# The vectorstore to use to index the child chunks
vectorstore = Chroma(
    collection_name="hypo-questions",
    embedding_function=OpenAIEmbeddings()
)

# The storage layer for the parent documents
store = InMemoryStore()
id_key = "doc_id"

# The retriever (empty to start)
retriever = MultiVectorRetriever(
    vectorstore=vectorstore, 
    docstore=store, 
    id_key=id_key,
)

doc_ids = [str(uuid.uuid4()) for _ in docs]

question_docs = []

for i, question_list in enumerate(hypothetical_questions):
    question_docs.extend([Document(page_content=s,metadata={id_key: doc_ids[i]}) for s in question_list])

retriever.vectorstore.add_documents(question_docs)
retriever.docstore.mset(list(zip(doc_ids, docs)))

sub_docs = vectorstore.similarity_search("justice breyer")

print(sub_docs)
```

結果:

```
    [Document(page_content="What is the President's stance on immigration reform?", metadata={'doc_id': '505d73e3-8350-46ec-a58e-3af032f04ab3'}),
     Document(page_content="What is the President's stance on immigration reform?", metadata={'doc_id': '1c9618f0-7660-4b4f-a37c-509cbbbf6dba'}),
     Document(page_content="What is the President's stance on immigration reform?", metadata={'doc_id': '82c08209-b904-46a8-9532-edd2380950b7'}),
     Document(page_content='What measures is the President proposing to protect the rights of LGBTQ+ Americans?', metadata={'doc_id': '82c08209-b904-46a8-9532-edd2380950b7'})]
```

```python
retrieved_docs = retriever.get_relevant_documents("justice breyer")

len(retrieved_docs[0].page_content)
```

結果:

```
    9194
```
