# Split code

`CodeTextSplitter` 允許您使用支援的多種程式語言來分割程式碼。導入枚舉 Language 並指定語言。

```python
from langchain.text_splitter import (
    RecursiveCharacterTextSplitter,
    Language,
)

# 支援語言的完整列表
[e.value for e in Language]
```

結果:

```python
[
    'cpp',
    'go',
    'java',
    'js',
    'php',
    'proto',
    'python',
    'rst',
    'ruby',
    'rust',
    'scala',
    'swift',
    'markdown',
    'latex',
    'html',
    'sol',]
```

```python
# 您也可以查看用於給定語言的分隔符
RecursiveCharacterTextSplitter.get_separators_for_language(Language.PYTHON)
```

結果:

```python
['\nclass ', '\ndef ', '\n\tdef ', '\n\n', '\n', ' ', '']
```

## Python

---

下面是使用 `PythonTextSplitter` 的範例：

```python
PYTHON_CODE = """
def hello_world():
    print("Hello, World!")

# Call the function
hello_world()
"""

python_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON, chunk_size=50, chunk_overlap=0
)

python_docs = python_splitter.create_documents([PYTHON_CODE])

print(python_docs)
```

結果:

```python
[Document(page_content='def hello_world():\n    print("Hello, World!")', metadata={}),
Document(page_content='# Call the function\nhello_world()', metadata={})]
```

## JS

---

以下是使用 JS 文字分割器的範例：

```python
JS_CODE = """
function helloWorld() {
  console.log("Hello, World!");
}

// Call the function
helloWorld();
"""

js_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.JS, chunk_size=60, chunk_overlap=0
)
js_docs = js_splitter.create_documents([JS_CODE])

print(js_docs)
```

結果:

```python
[Document(page_content='function helloWorld() {\n  console.log("Hello, World!");\n}', metadata={}),
    Document(page_content='// Call the function\nhelloWorld();', metadata={})]
```

## Markdown

---

以下是使用 Markdown 文字分割器的範例：

```python
markdown_text = """
# 🦜️🔗 LangChain

⚡ Building applications with LLMs through composability ⚡

## Quick Install

```bash
# Hopefully this code block isn't split
pip install langchain
```

As an open source project in a rapidly developing field, we are extremely open to contributions.
"""

md_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.MARKDOWN, chunk_size=60, chunk_overlap=0
)

md_docs = md_splitter.create_documents([markdown_text])

print(md_docs)
```

結果:

```python
[Document(page_content='# 🦜️🔗 LangChain', metadata={}),
    Document(page_content='⚡ Building applications with LLMs through composability ⚡', metadata={}),
    Document(page_content='## Quick Install', metadata={}),
    Document(page_content="```bash\n# Hopefully this code block isn't split", metadata={}),
    Document(page_content='pip install langchain', metadata={}),
    Document(page_content='```', metadata={}),
    Document(page_content='As an open source project in a rapidly developing field, we', metadata={}),
    Document(page_content='are extremely open to contributions.', metadata={})]
```

## HTML

---

以下是使用 HTML 文字分割器的範例：

```python
html_text = """
<!DOCTYPE html>
<html>
    <head>
        <title>🦜️🔗 LangChain</title>
        <style>
            body {
                font-family: Arial, sans-serif;
            }
            h1 {
                color: darkblue;
            }
        </style>
    </head>
    <body>
        <div>
            <h1>🦜️🔗 LangChain</h1>
            <p>⚡ Building applications with LLMs through composability ⚡</p>
        </div>
        <div>
            As an open source project in a rapidly developing field, we are extremely open to contributions.
        </div>
    </body>
</html>
"""

html_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.HTML, chunk_size=60, chunk_overlap=0
)

html_docs = html_splitter.create_documents([html_text])

print(html_docs)
```

結果:

```python
[Document(page_content='<!DOCTYPE html>\n<html>', metadata={}),
    Document(page_content='<head>\n        <title>🦜️🔗 LangChain</title>', metadata={}),
    Document(page_content='<style>\n            body {\n                font-family: Aria', metadata={}),
    Document(page_content='l, sans-serif;\n            }\n            h1 {', metadata={}),
    Document(page_content='color: darkblue;\n            }\n        </style>\n    </head', metadata={}),
    Document(page_content='>', metadata={}),
    Document(page_content='<body>', metadata={}),
    Document(page_content='<div>\n            <h1>🦜️🔗 LangChain</h1>', metadata={}),
    Document(page_content='<p>⚡ Building applications with LLMs through composability ⚡', metadata={}),
    Document(page_content='</p>\n        </div>', metadata={}),
    Document(page_content='<div>\n            As an open source project in a rapidly dev', metadata={}),
    Document(page_content='eloping field, we are extremely open to contributions.', metadata={}),
    Document(page_content='</div>\n    </body>\n</html>', metadata={})]
```

## Solidity

---

以下是使用 Solidity 文字分割器的範例：

```python
SOL_CODE = """
pragma solidity ^0.8.20;
contract HelloWorld {
   function add(uint a, uint b) pure public returns(uint) {
       return a + b;
   }
}
"""

sol_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.SOL, chunk_size=128, chunk_overlap=0
)
sol_docs = sol_splitter.create_documents([SOL_CODE])

print(sol_docs)
```

結果:

```python
[
    Document(page_content='pragma solidity ^0.8.20;', metadata={}),
    Document(page_content='contract HelloWorld {\n   function add(uint a, uint b) pure public returns(uint) {\n       return a + b;\n   }\n}', metadata={})
]
```


