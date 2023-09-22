# PDF

> [Portable Document Format (PDF)](https://en.wikipedia.org/wiki/PDF)，標準化為ISO 32000，是 AdAdobe 於 1992 年開發的一種文件格式，用於以獨立於應用程式軟體、硬體和作業系統的方式呈現文檔，包括文字格式和圖像。

本文涵蓋瞭解如何將 PDF 文件載入為我們下游應用程式使用的文件格式。

## 使用 PyPDF

使用 `pypdf` 將 PDF 載入到成 [document](https://api.python.langchain.com/en/latest/schema/langchain.schema.document.Document.html?highlight=document#langchain.schema.document.Document) 陣列，其中每個 `document` 包含頁面內容和帶有 `page` 的元資料。

```python
pip install pypdf

from langchain.document_loaders import PyPDFLoader

loader = PyPDFLoader("example_data/layout-parser-paper.pdf")

pages = loader.load_and_split()

print(pages[0])
```

結果:

```python
Document(page_content='LayoutParser : A Uni\x0ced Toolkit for Deep\nLearning Based Document Image Analysis\nZejiang Shen1( \x00), Ruochen Zhang2, Melissa Dell3, Benjamin Charles Germain\nLee4, Jacob Carlson3, and Weining Li5\n1Allen Institute for AI\nshannons@allenai.org\n2Brown University\nruochen zhang@brown.edu\n3Harvard University\nfmelissadell,jacob carlson g@fas.harvard.edu\n4University of Washington\nbcgl@cs.washington.edu\n5University of Waterloo\nw422li@uwaterloo.ca\nAbstract. Recent advances in document image analysis (DIA) have been\nprimarily driven by the application of neural networks. Ideally, research\noutcomes could be easily deployed in production and extended for further\ninvestigation. However, various factors like loosely organized codebases\nand sophisticated model con\x0cgurations complicate the easy reuse of im-\nportant innovations by a wide audience. Though there have been on-going\ne\x0borts to improve reusability and simplify deep learning (DL) model\ndevelopment in disciplines like natural language processing and computer\nvision, none of them are optimized for challenges in the domain of DIA.\nThis represents a major gap in the existing toolkit, as DIA is central to\nacademic research across a wide range of disciplines in the social sciences\nand humanities. This paper introduces LayoutParser , an open-source\nlibrary for streamlining the usage of DL in DIA research and applica-\ntions. The core LayoutParser library comes with a set of simple and\nintuitive interfaces for applying and customizing DL models for layout de-\ntection, character recognition, and many other document processing tasks.\nTo promote extensibility, LayoutParser also incorporates a community\nplatform for sharing both pre-trained models and full document digiti-\nzation pipelines. We demonstrate that LayoutParser is helpful for both\nlightweight and large-scale digitization pipelines in real-word use cases.\nThe library is publicly available at https://layout-parser.github.io .\nKeywords: Document Image Analysis ·Deep Learning ·Layout Analysis\n·Character Recognition ·Open Source library ·Toolkit.\n1 Introduction\nDeep Learning(DL)-based approaches are the state-of-the-art for a wide range of\ndocument image analysis (DIA) tasks including document image classi\x0ccation [ 11,arXiv:2103.15348v2  [cs.CV]  21 Jun 2021', metadata={'source': 'example_data/layout-parser-paper.pdf', 'page': 0})
```

這種方法的優點是可以使用頁碼檢索文件。

我們想要使用 `OpenAIEmbeddings`，因此我們必須取得 OpenAI API 金鑰。

```python
import os
import getpass

os.environ['OPENAI_API_KEY'] = getpass.getpass('OpenAI API Key:')
```

輸入 OPENAI API 金鑰:

```python
OpenAI API Key: ········
```

```python
from langchain.vectorstores import FAISS
from langchain.embeddings.openai import OpenAIEmbeddings

faiss_index = FAISS.from_documents(pages, OpenAIEmbeddings())

docs = faiss_index.similarity_search("How will the community be engaged?", k=2)

for doc in docs:
    print(str(doc.metadata["page"]) + ":", doc.page_content[:300])
```


結果:

```python
9: 10 Z. Shen et al.
Fig. 4: Illustration of (a) the original historical Japanese document with layout
detection results and (b) a recreated version of the document image that achieves
much better character recognition recall. The reorganization algorithm rearranges
the tokens based on the their detect
3: 4 Z. Shen et al.
Efficient Data AnnotationC u s t o m i z e d  M o d e l  T r a i n i n gModel Cust omizationDI A Model HubDI A Pipeline SharingCommunity PlatformLa y out Detection ModelsDocument Images 
T h e  C o r e  L a y o u t P a r s e r  L i b r a r yOCR ModuleSt or age & VisualizationLa y ou
```

## 使用 MathPix

靈感來自 Daniel Gross 的 [danielgross/mathpix2gpt.py](https://gist.github.com/danielgross/3ab4104e14faccc12b49200843adab21) 的 gist。

詳請見: [MathPix](https://mathpix.com/)

```python
from langchain.document_loaders import MathpixPDFLoader

loader = MathpixPDFLoader("example_data/layout-parser-paper.pdf")

loader = MathpixPDFLoader("example_data/layout-parser-paper.pdf")
```

## 使用 Unstructured

```python
from langchain.document_loaders import UnstructuredPDFLoader

loader = UnstructuredPDFLoader("example_data/layout-parser-paper.pdf")

data = loader.load()
```

### Retain Elements

在背後，`Unstructured` 為不同的文字區塊創建不同的 elements。預設情況下，我們將它們組合在一起，但您可以透過指定 `mode="elements"` 輕鬆保持這種分離。

```python
loader = UnstructuredPDFLoader("example_data/layout-parser-paper.pdf", mode="elements")

data = loader.load()

print(data[0])
```

結果:

```python
    Document(page_content='LayoutParser: A Uniﬁed Toolkit for Deep\nLearning Based Document Image Analysis\nZejiang Shen1 (�), Ruochen Zhang2, Melissa Dell3, Benjamin Charles Germain\nLee4, Jacob Carlson3, and Weining Li5\n1 Allen Institute for AI\nshannons@allenai.org\n2 Brown University\nruochen zhang@brown.edu\n3 Harvard University\n{melissadell,jacob carlson}@fas.harvard.edu\n4 University of Washington\nbcgl@cs.washington.edu\n5 University of Waterloo\nw422li@uwaterloo.ca\nAbstract. Recent advances in document image analysis (DIA) have been\nprimarily driven by the application of neural networks. Ideally, research\noutcomes could be easily deployed in production and extended for further\ninvestigation. However, various factors like loosely organized codebases\nand sophisticated model conﬁgurations complicate the easy reuse of im-\nportant innovations by a wide audience. Though there have been on-going\neﬀorts to improve reusability and simplify deep learning (DL) model\ndevelopment in disciplines like natural language processing and computer\nvision, none of them are optimized for challenges in the domain of DIA.\nThis represents a major gap in the existing toolkit, as DIA is central to\nacademic research across a wide range of disciplines in the social sciences\nand humanities. This paper introduces LayoutParser, an open-source\nlibrary for streamlining the usage of DL in DIA research and applica-\ntions. The core LayoutParser library comes with a set of simple and\nintuitive interfaces for applying and customizing DL models for layout de-\ntection, character recognition, and many other document processing tasks.\nTo promote extensibility, LayoutParser also incorporates a community\nplatform for sharing both pre-trained models and full document digiti-\nzation pipelines. We demonstrate that LayoutParser is helpful for both\nlightweight and large-scale digitization pipelines in real-word use cases.\nThe library is publicly available at https://layout-parser.github.io.\nKeywords: Document Image Analysis · Deep Learning · Layout Analysis\n· Character Recognition · Open Source library · Toolkit.\n1\nIntroduction\nDeep Learning(DL)-based approaches are the state-of-the-art for a wide range of\ndocument image analysis (DIA) tasks including document image classiﬁcation [11,\narXiv:2103.15348v2  [cs.CV]  21 Jun 2021\n', lookup_str='', metadata={'file_path': 'example_data/layout-parser-paper.pdf', 'page_number': 1, 'total_pages': 16, 'format': 'PDF 1.5', 'title': '', 'author': '', 'subject': '', 'keywords': '', 'creator': 'LaTeX with hyperref', 'producer': 'pdfTeX-1.40.21', 'creationDate': 'D:20210622012710Z', 'modDate': 'D:20210622012710Z', 'trapped': '', 'encryption': None}, lookup_index=0)
```

### 獲取遠端 PDF

這涵蓋瞭如何將線上 PDF 載入為我們可以在下游使用的文件格式。這可用於各種線上 PDF 網站，例如 https://open.umn.edu/opentextbooks/textbooks/ 和 https://arxiv.org/archive/

!!! info
    注意：所有其他 PDF 載入器也可用於取得遠端 PDF，但 `OnlinePDFLoader` 是一個 legacy 函數，專門與 `UnstructedPDFLoader` 搭配使用。

```python
from langchain.document_loaders import OnlinePDFLoader

loader = OnlinePDFLoader("https://arxiv.org/pdf/2302.03803.pdf")

data = loader.load()

print(data)
```

## 使用 PyPDFium2

```python
from langchain.document_loaders import PyPDFium2Loader

loader = PyPDFium2Loader("example_data/layout-parser-paper.pdf")

data = loader.load()
```

## 使用 PDFMiner

```python
from langchain.document_loaders import PDFMinerLoader

from langchain.document_loaders import PDFMinerLoader

data = loader.load()
```

### 使用 PDFMiner 產生 HTML

這對於將文字在語義上分塊很有幫助，因為輸出的 html 內容可以透過 `BeautifulSoup` 進行解析，以獲得有關字體大小、頁碼、PDF 頁眉/頁腳等的更多結構化和豐富的資訊。

```python
from langchain.document_loaders import PDFMinerPDFasHTMLLoader

loader = PDFMinerPDFasHTMLLoader("example_data/layout-parser-paper.pdf")

data = loader.load()[0]   # entire PDF is loaded as a single Document

from bs4 import BeautifulSoup
soup = BeautifulSoup(data.page_content,'html.parser')
content = soup.find_all('div')

import re
cur_fs = None
cur_text = ''
snippets = []   # first collect all snippets that have the same font size
for c in content:
    sp = c.find('span')
    if not sp:
        continue
    st = sp.get('style')
    if not st:
        continue
    fs = re.findall('font-size:(\d+)px',st)
    if not fs:
        continue
    fs = int(fs[0])
    if not cur_fs:
        cur_fs = fs
    if fs == cur_fs:
        cur_text += c.text
    else:
        snippets.append((cur_text,cur_fs))
        cur_fs = fs
        cur_text = c.text
snippets.append((cur_text,cur_fs))
# Note: The above logic is very straightforward. One can also add more strategies such as removing duplicate snippets (as
# headers/footers in a PDF appear on multiple pages so if we find duplicates it's safe to assume that it is redundant info)

from langchain.docstore.document import Document
cur_idx = -1
semantic_snippets = []
# Assumption: headings have higher font size than their respective content
for s in snippets:
    # if current snippet's font size > previous section's heading => it is a new heading
    if not semantic_snippets or s[1] > semantic_snippets[cur_idx].metadata['heading_font']:
        metadata={'heading':s[0], 'content_font': 0, 'heading_font': s[1]}
        metadata.update(data.metadata)
        semantic_snippets.append(Document(page_content='',metadata=metadata))
        cur_idx += 1
        continue
    
    # if current snippet's font size <= previous section's content => content belongs to the same section (one can also create
    # a tree like structure for sub sections if needed but that may require some more thinking and may be data specific)
    if not semantic_snippets[cur_idx].metadata['content_font'] or s[1] <= semantic_snippets[cur_idx].metadata['content_font']:
        semantic_snippets[cur_idx].page_content += s[0]
        semantic_snippets[cur_idx].metadata['content_font'] = max(s[1], semantic_snippets[cur_idx].metadata['content_font'])
        continue
    
    # if current snippet's font size > previous section's content but less than previous section's heading than also make a new 
    # section (e.g. title of a PDF will have the highest font size but we don't want it to subsume all sections)
    metadata={'heading':s[0], 'content_font': 0, 'heading_font': s[1]}
    metadata.update(data.metadata)
    semantic_snippets.append(Document(page_content='',metadata=metadata))
    cur_idx += 1
```

## 使用 PyMuPDF

這是最快的 PDF 解析選項，包含有關 PDF 及其頁面的詳細元數據，並且每頁返回一個文件。

```python
from langchain.document_loaders import PyMuPDFLoader

loader = PyMuPDFLoader("example_data/layout-parser-paper.pdf")

data = loader.load()

print(data[0])
```

結果:

```python
    Document(page_content='LayoutParser: A Uniﬁed Toolkit for Deep\nLearning Based Document Image Analysis\nZejiang Shen1 (�), Ruochen Zhang2, Melissa Dell3, Benjamin Charles Germain\nLee4, Jacob Carlson3, and Weining Li5\n1 Allen Institute for AI\nshannons@allenai.org\n2 Brown University\nruochen zhang@brown.edu\n3 Harvard University\n{melissadell,jacob carlson}@fas.harvard.edu\n4 University of Washington\nbcgl@cs.washington.edu\n5 University of Waterloo\nw422li@uwaterloo.ca\nAbstract. Recent advances in document image analysis (DIA) have been\nprimarily driven by the application of neural networks. Ideally, research\noutcomes could be easily deployed in production and extended for further\ninvestigation. However, various factors like loosely organized codebases\nand sophisticated model conﬁgurations complicate the easy reuse of im-\nportant innovations by a wide audience. Though there have been on-going\neﬀorts to improve reusability and simplify deep learning (DL) model\ndevelopment in disciplines like natural language processing and computer\nvision, none of them are optimized for challenges in the domain of DIA.\nThis represents a major gap in the existing toolkit, as DIA is central to\nacademic research across a wide range of disciplines in the social sciences\nand humanities. This paper introduces LayoutParser, an open-source\nlibrary for streamlining the usage of DL in DIA research and applica-\ntions. The core LayoutParser library comes with a set of simple and\nintuitive interfaces for applying and customizing DL models for layout de-\ntection, character recognition, and many other document processing tasks.\nTo promote extensibility, LayoutParser also incorporates a community\nplatform for sharing both pre-trained models and full document digiti-\nzation pipelines. We demonstrate that LayoutParser is helpful for both\nlightweight and large-scale digitization pipelines in real-word use cases.\nThe library is publicly available at https://layout-parser.github.io.\nKeywords: Document Image Analysis · Deep Learning · Layout Analysis\n· Character Recognition · Open Source library · Toolkit.\n1\nIntroduction\nDeep Learning(DL)-based approaches are the state-of-the-art for a wide range of\ndocument image analysis (DIA) tasks including document image classiﬁcation [11,\narXiv:2103.15348v2  [cs.CV]  21 Jun 2021\n', lookup_str='', metadata={'file_path': 'example_data/layout-parser-paper.pdf', 'page_number': 1, 'total_pages': 16, 'format': 'PDF 1.5', 'title': '', 'author': '', 'subject': '', 'keywords': '', 'creator': 'LaTeX with hyperref', 'producer': 'pdfTeX-1.40.21', 'creationDate': 'D:20210622012710Z', 'modDate': 'D:20210622012710Z', 'trapped': '', 'encryption': None}, lookup_index=0)
```

此外，您可以在載入呼叫中將 PyMuPDF 文件中的任何選項作為關鍵字參數傳遞，並將其傳遞給 `get_text()` 呼叫。

## PyPDF 目錄

從目錄載入 PDF

```python
from langchain.document_loaders import PyPDFDirectoryLoader

loader = PyPDFDirectoryLoader("example_data/")

docs = loader.load()
```

## 使用 PDFPlumber

與 `PyMuPDF` 一樣，輸出文件包含有關 PDF 及其頁面的詳細元數據，並每頁傳回一個 `document` 物件。

```python
from langchain.document_loaders import PDFPlumberLoader

loader = PDFPlumberLoader("example_data/layout-parser-paper.pdf")

data = loader.load()

print(data[0])
```

結果:

```python
    Document(page_content='LayoutParser: A Unified Toolkit for Deep\nLearning Based Document Image Analysis\nZejiang Shen1 ((cid:0)), Ruochen Zhang2, Melissa Dell3, Benjamin Charles Germain\nLee4, Jacob Carlson3, and Weining Li5\n1 Allen Institute for AI\n1202 shannons@allenai.org\n2 Brown University\nruochen zhang@brown.edu\n3 Harvard University\nnuJ {melissadell,jacob carlson}@fas.harvard.edu\n4 University of Washington\nbcgl@cs.washington.edu\n12 5 University of Waterloo\nw422li@uwaterloo.ca\n]VC.sc[\nAbstract. Recentadvancesindocumentimageanalysis(DIA)havebeen\nprimarily driven by the application of neural networks. Ideally, research\noutcomescouldbeeasilydeployedinproductionandextendedforfurther\ninvestigation. However, various factors like loosely organized codebases\nand sophisticated model configurations complicate the easy reuse of im-\n2v84351.3012:viXra portantinnovationsbyawideaudience.Thoughtherehavebeenon-going\nefforts to improve reusability and simplify deep learning (DL) model\ndevelopmentindisciplineslikenaturallanguageprocessingandcomputer\nvision, none of them are optimized for challenges in the domain of DIA.\nThis represents a major gap in the existing toolkit, as DIA is central to\nacademicresearchacross awiderangeof disciplinesinthesocialsciences\nand humanities. This paper introduces LayoutParser, an open-source\nlibrary for streamlining the usage of DL in DIA research and applica-\ntions. The core LayoutParser library comes with a set of simple and\nintuitiveinterfacesforapplyingandcustomizingDLmodelsforlayoutde-\ntection,characterrecognition,andmanyotherdocumentprocessingtasks.\nTo promote extensibility, LayoutParser also incorporates a community\nplatform for sharing both pre-trained models and full document digiti-\nzation pipelines. We demonstrate that LayoutParser is helpful for both\nlightweight and large-scale digitization pipelines in real-word use cases.\nThe library is publicly available at https://layout-parser.github.io.\nKeywords: DocumentImageAnalysis·DeepLearning·LayoutAnalysis\n· Character Recognition · Open Source library · Toolkit.\n1 Introduction\nDeep Learning(DL)-based approaches are the state-of-the-art for a wide range of\ndocumentimageanalysis(DIA)tasksincludingdocumentimageclassification[11,', metadata={'source': 'example_data/layout-parser-paper.pdf', 'file_path': 'example_data/layout-parser-paper.pdf', 'page': 1, 'total_pages': 16, 'Author': '', 'CreationDate': 'D:20210622012710Z', 'Creator': 'LaTeX with hyperref', 'Keywords': '', 'ModDate': 'D:20210622012710Z', 'PTEX.Fullbanner': 'This is pdfTeX, Version 3.14159265-2.6-1.40.21 (TeX Live 2020) kpathsea version 6.3.2', 'Producer': 'pdfTeX-1.40.21', 'Subject': '', 'Title': '', 'Trapped': 'False'})
```

## 使用 AmazonTextractPDFParser

AmazonTextractPDFLoader 呼叫 [Amazon Textract Service](https://aws.amazon.com/textract/) 將 PDF 轉換為文件結構。該載入程式目前執行純 OCR，並根據需求計劃提供更多功能，例如佈局支援。支援最多 3000 頁和 512 MB 大小的單頁和多頁文件。

為了使呼叫成功，需要一個 AWS 帳戶，類似於 AWS CLI 要求。

除了 AWS 配置之外，它與其他 PDF 載入器非常相似，同時還支援 JPEG、PNG 和 TIFF 以及非原生 PDF 格式。

```python
from langchain.document_loaders import AmazonTextractPDFLoader

loader = AmazonTextractPDFLoader("example_data/alejandro_rosalez_sample-small.jpeg")

documents = loader.load()
```

