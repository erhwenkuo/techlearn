# File Directory

本文涵蓋瞭解如何載入特定目錄中的所有文件。

在底層，預設使用 [UnstructedLoader](https://python.langchain.com/docs/integrations/document_loaders/unstructured_file.html)。

```python
from langchain.document_loaders import DirectoryLoader
```

我們可以使用 `glob` 參數來控制載入哪些檔案。請注意，這裡它不會載入 `.rst` 檔案或 `.html` 檔案。

```python
loader = DirectoryLoader('../', glob="**/*.md")

docs = loader.load()
```

!!! tips
    [UnstructedLoader](https://python.langchain.com/docs/integrations/document_loaders/unstructured_file.html) 目前支援載入文字檔案、powerpoints、html、pdf、圖片等等類型檔案。

## 顯示進度條

預設不會顯示 progress bar。若要顯示 progress bar，請安裝 `tqdm` 函式庫（例如 `pip install tqdm`），並將 `show_progress` 參數設為 `True`。

```python
loader = DirectoryLoader('../', glob="**/*.md", show_progress=True)

docs = loader.load()
```

範例結果:

```
Requirement already satisfied: tqdm in /Users/jon/.pyenv/versions/3.9.16/envs/microbiome-app/lib/python3.9/site-packages (4.65.0)


0it [00:00, ?it/s]
```

## 使用多線程

預設情況下，載入發生在一個執行緒中。為了利用多個線程，請將 `use_multithreading` 旗標設為 `true`。

```python
loader = DirectoryLoader('../', glob="**/*.md", use_multithreading=True)

docs = loader.load()
```

## 更改 loader class

預設情況下，這使用  [UnstructedLoader](https://python.langchain.com/docs/integrations/document_loaders/unstructured_file.html) 類別。但是，您可以輕鬆地更改 loader 的類型。

```python
from langchain.document_loaders import TextLoader

loader = DirectoryLoader('../', glob="**/*.md", loader_cls=TextLoader)

docs = loader.load()
```

如果需要載入 Python 原始碼文件，請使用 `PythonLoader`。

```python
from langchain.document_loaders import PythonLoader

loader = DirectoryLoader('../../../../../', glob="**/*.py", loader_cls=PythonLoader)

docs = loader.load()
```

## TextLoader 自動偵測檔案編碼

在此範例中，我們將看到一些策略，這些策略在使用 [`TextLoader`](https://api.python.langchain.com/en/latest/document_loaders/langchain.document_loaders.text.TextLoader.html?highlight=textloader#) 類別從目錄載入大量任意檔案時非常有用。

首先為了說明問題，讓我們嘗試載入具有任意編碼的多個文字。

```python
path = '../../../../../tests/integration_tests/examples'

loader = DirectoryLoader(path, glob="**/*.txt", loader_cls=TextLoader)
```

### A. Default Behavior

```python
loader.load()
```

```html
<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #800000; text-decoration-color: #800000">╭─────────────────────────────── </span><span style="color: #800000; text-decoration-color: #800000; font-weight: bold">Traceback </span><span style="color: #bf7f7f; text-decoration-color: #bf7f7f; font-weight: bold">(most recent call last)</span><span style="color: #800000; text-decoration-color: #800000"> ────────────────────────────────╮</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #bfbf7f; text-decoration-color: #bfbf7f">/data/source/langchain/langchain/document_loaders/</span><span style="color: #808000; text-decoration-color: #808000; font-weight: bold">text.py</span>:<span style="color: #0000ff; text-decoration-color: #0000ff">29</span> in <span style="color: #00ff00; text-decoration-color: #00ff00">load</span>                             <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">26 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   </span>text = <span style="color: #808000; text-decoration-color: #808000">""</span>                                                                           <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">27 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">with</span> <span style="color: #00ffff; text-decoration-color: #00ffff">open</span>(<span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.file_path, encoding=<span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.encoding) <span style="color: #0000ff; text-decoration-color: #0000ff">as</span> f:                             <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">28 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">try</span>:                                                                            <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #800000; text-decoration-color: #800000">❱ </span>29 <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   </span>text = f.read()                                                             <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">30 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">except</span> <span style="color: #00ffff; text-decoration-color: #00ffff">UnicodeDecodeError</span> <span style="color: #0000ff; text-decoration-color: #0000ff">as</span> e:                                                 <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">31 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">if</span> <span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.autodetect_encoding:                                                <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">32 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   </span>detected_encodings = <span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.detect_file_encodings()                       <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #bfbf7f; text-decoration-color: #bfbf7f">/home/spike/.pyenv/versions/3.9.11/lib/python3.9/</span><span style="color: #808000; text-decoration-color: #808000; font-weight: bold">codecs.py</span>:<span style="color: #0000ff; text-decoration-color: #0000ff">322</span> in <span style="color: #00ff00; text-decoration-color: #00ff00">decode</span>                         <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f"> 319 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   </span><span style="color: #0000ff; text-decoration-color: #0000ff">def</span> <span style="color: #00ff00; text-decoration-color: #00ff00">decode</span>(<span style="color: #00ffff; text-decoration-color: #00ffff">self</span>, <span style="color: #00ffff; text-decoration-color: #00ffff">input</span>, final=<span style="color: #0000ff; text-decoration-color: #0000ff">False</span>):                                                 <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f"> 320 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f"># decode input (taking the buffer into account)</span>                                   <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f"> 321 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   </span>data = <span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.buffer + <span style="color: #00ffff; text-decoration-color: #00ffff">input</span>                                                        <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #800000; text-decoration-color: #800000">❱ </span> 322 <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   </span>(result, consumed) = <span style="color: #00ffff; text-decoration-color: #00ffff">self</span>._buffer_decode(data, <span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.errors, final)                <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f"> 323 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f"># keep undecoded input until the next call</span>                                        <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f"> 324 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   </span><span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.buffer = data[consumed:]                                                     <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f"> 325 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">return</span> result                                                                     <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">╰──────────────────────────────────────────────────────────────────────────────────────────────────╯</span>
<span style="color: #ff0000; text-decoration-color: #ff0000; font-weight: bold">UnicodeDecodeError: </span><span style="color: #008000; text-decoration-color: #008000">'utf-8'</span> codec can't decode byte <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0xca</span> in position <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0</span>: invalid continuation byte

<span style="font-style: italic">The above exception was the direct cause of the following exception:</span>

<span style="color: #800000; text-decoration-color: #800000">╭─────────────────────────────── </span><span style="color: #800000; text-decoration-color: #800000; font-weight: bold">Traceback </span><span style="color: #bf7f7f; text-decoration-color: #bf7f7f; font-weight: bold">(most recent call last)</span><span style="color: #800000; text-decoration-color: #800000"> ────────────────────────────────╮</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> in <span style="color: #00ff00; text-decoration-color: #00ff00">&lt;module&gt;</span>:<span style="color: #0000ff; text-decoration-color: #0000ff">1</span>                                                                                    <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #800000; text-decoration-color: #800000">❱ </span>1 loader.load()                                                                                <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">2 </span>                                                                                             <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #bfbf7f; text-decoration-color: #bfbf7f">/data/source/langchain/langchain/document_loaders/</span><span style="color: #808000; text-decoration-color: #808000; font-weight: bold">directory.py</span>:<span style="color: #0000ff; text-decoration-color: #0000ff">84</span> in <span style="color: #00ff00; text-decoration-color: #00ff00">load</span>                        <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">81 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">if</span> <span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.silent_errors:                                              <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">82 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   │   </span>logger.warning(e)                                               <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">83 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">else</span>:                                                               <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #800000; text-decoration-color: #800000">❱ </span>84 <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">raise</span> e                                                         <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">85 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">finally</span>:                                                                <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">86 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">if</span> pbar:                                                            <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">87 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   │   </span>pbar.update(<span style="color: #0000ff; text-decoration-color: #0000ff">1</span>)                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #bfbf7f; text-decoration-color: #bfbf7f">/data/source/langchain/langchain/document_loaders/</span><span style="color: #808000; text-decoration-color: #808000; font-weight: bold">directory.py</span>:<span style="color: #0000ff; text-decoration-color: #0000ff">78</span> in <span style="color: #00ff00; text-decoration-color: #00ff00">load</span>                        <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">75 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">if</span> i.is_file():                                                                 <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">76 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">if</span> _is_visible(i.relative_to(p)) <span style="color: #ff00ff; text-decoration-color: #ff00ff">or</span> <span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.load_hidden:                       <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">77 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">try</span>:                                                                    <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #800000; text-decoration-color: #800000">❱ </span>78 <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   </span>sub_docs = <span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.loader_cls(<span style="color: #00ffff; text-decoration-color: #00ffff">str</span>(i), **<span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.loader_kwargs).load()     <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">79 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   </span>docs.extend(sub_docs)                                               <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">80 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">except</span> <span style="color: #00ffff; text-decoration-color: #00ffff">Exception</span> <span style="color: #0000ff; text-decoration-color: #0000ff">as</span> e:                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">81 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">if</span> <span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.silent_errors:                                              <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #bfbf7f; text-decoration-color: #bfbf7f">/data/source/langchain/langchain/document_loaders/</span><span style="color: #808000; text-decoration-color: #808000; font-weight: bold">text.py</span>:<span style="color: #0000ff; text-decoration-color: #0000ff">44</span> in <span style="color: #00ff00; text-decoration-color: #00ff00">load</span>                             <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>                                                                                                  <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">41 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">except</span> <span style="color: #00ffff; text-decoration-color: #00ffff">UnicodeDecodeError</span>:                                          <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">42 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">continue</span>                                                        <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">43 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">else</span>:                                                                       <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span> <span style="color: #800000; text-decoration-color: #800000">❱ </span>44 <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">raise</span> <span style="color: #00ffff; text-decoration-color: #00ffff">RuntimeError</span>(<span style="color: #808000; text-decoration-color: #808000">f"Error loading {</span><span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.file_path<span style="color: #808000; text-decoration-color: #808000">}"</span>) <span style="color: #0000ff; text-decoration-color: #0000ff">from</span> <span style="color: #00ffff; text-decoration-color: #00ffff; text-decoration: underline">e</span>            <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">45 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">except</span> <span style="color: #00ffff; text-decoration-color: #00ffff">Exception</span> <span style="color: #0000ff; text-decoration-color: #0000ff">as</span> e:                                                          <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">46 </span><span style="color: #7f7f7f; text-decoration-color: #7f7f7f">│   │   │   │   </span><span style="color: #0000ff; text-decoration-color: #0000ff">raise</span> <span style="color: #00ffff; text-decoration-color: #00ffff">RuntimeError</span>(<span style="color: #808000; text-decoration-color: #808000">f"Error loading {</span><span style="color: #00ffff; text-decoration-color: #00ffff">self</span>.file_path<span style="color: #808000; text-decoration-color: #808000">}"</span>) <span style="color: #0000ff; text-decoration-color: #0000ff">from</span> <span style="color: #00ffff; text-decoration-color: #00ffff; text-decoration: underline">e</span>                <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">│</span>   <span style="color: #7f7f7f; text-decoration-color: #7f7f7f">47 </span>                                                                                            <span style="color: #800000; text-decoration-color: #800000">│</span>
<span style="color: #800000; text-decoration-color: #800000">╰──────────────────────────────────────────────────────────────────────────────────────────────────╯</span>
<span style="color: #ff0000; text-decoration-color: #ff0000; font-weight: bold">RuntimeError: </span>Error loading ..<span style="color: #800080; text-decoration-color: #800080">/../../../../tests/integration_tests/examples/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">example-non-utf8.txt</span>
</pre>
```

檔案 `example-non-utf8.txt` 使用不同的編碼，因此 `load()` 函數失敗，並顯示一個有用的訊息，指示哪個檔案解碼失敗。

在 `TextLoader` 的預設行為下，任何文件載入失敗都會導致整個載入過程失敗，並且不會載入任何文件。

### B. Silent fail

我們可以將參數 `silent_errors` 傳遞給 `DirectoryLoader` 來跳過無法載入的檔案並繼續載入過程。

```python
loader = DirectoryLoader(path, glob="**/*.txt", loader_cls=TextLoader, silent_errors=True)

docs = loader.load()
```

有錯誤的訊息:

```
Error loading ../../../../../tests/integration_tests/examples/example-non-utf8.txt
```

檢查:

```python
doc_sources = [doc.metadata['source']  for doc in docs]

doc_sources
```

結果:

```
['../../../../../tests/integration_tests/examples/whatsapp_chat.txt',
    '../../../../../tests/integration_tests/examples/example-utf8.txt']
```

### C. Auto detect encodings

我們也可以透過將 `autodetect_encoding` 傳遞給 loader 類別，要求 `TextLoader` 在失敗之前自動偵測檔案編碼。

```python
text_loader_kwargs={'autodetect_encoding': True}

loader = DirectoryLoader(path, glob="**/*.txt", loader_cls=TextLoader, loader_kwargs=text_loader_kwargs)

docs = loader.load()
```

檢查:

```python
doc_sources = [doc.metadata['source']  for doc in docs]

doc_sources
```

結果:

```python
['../../../../../tests/integration_tests/examples/example-non-utf8.txt',
    '../../../../../tests/integration_tests/examples/whatsapp_chat.txt',
    '../../../../../tests/integration_tests/examples/example-utf8.txt']
```