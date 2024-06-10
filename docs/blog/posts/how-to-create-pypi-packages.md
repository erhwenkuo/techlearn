---
draft: false 
date: 2024-06-09
categories:
  - Python
  - Packaging
  - PyPI
---

# 建立 Python 套件並將其發佈到 PyPI

![](./create-first-python-package/how-to-create-pypi-packages.avif)

PyPI 是 [Python Package](https://www.turing.com/kb/7-ways-to-include-non-python-files-into-python-package) Index 的縮寫，是 Python 程式語言的官方第三方套件儲存庫。它是一個中央儲存庫，託管和分發軟體包供 Python 開發人員使用。

開發人員可以將 Python 套件上傳到 PyPI，其他人可以輕鬆存取這些套件。這些套件可以包括函式庫、框架、實用程式和工具來擴展和增強 Python 的功能。

本文將介紹 PyPI 的優點、套件的元件以及如何建立 PyPI 套件。

## 什麼是 PyPI?

PyPI 用於尋找和安裝專案的套件。它允許開發人員輕鬆管理依賴項並簡化應用程式的安裝過程。借助 PyPI，他們可以透過重複使用現有程式碼而不是從頭開始編寫所有內容來節省時間和精力。

PyPI 由 Python 軟體基金會 (PSF) 維護，可供開發人員和最終用戶免費使用。它是 Python 生態系統的重要組成部分，為該語言的流行和發展做出了貢獻。

在 Python 中，套件是一種分層組織相關模組、函數和類別的方式。它提供了一種捆綁程式碼的結構化方法，可以將其視為包含一個或多個 Python 模組的目錄。它有助於組織程式碼並使其更易於管理和分發。

Package 通常包含一個 `init.py` 文件，該文件在導入套件時執行。它可以定義包級屬性、函數和類，可供套件中的其他模組存取。 `init.py` 檔案也可以包含導入語句以從其他套件導入模組和函數。

可以使用 pip 等套件管理器來安裝和管理 Python 套件，pip 可以自動下載並安裝所需的依賴項。

透過套件，開發人員可以將相關功能封裝到一個單元中，從而更容易重複使用和維護程式碼。例如，流行的 NumPy 套件提供了處理數組和矩陣的功能，而 Pandas 套件提供了用於資料分析和操作的工具。

<!-- more -->

## PyPI: Python 專案的支柱

PyPI 擁有大量可以安裝並整合到 Python 專案中的程式庫、框架和工具。以下是一些好處：

1. **Easy package discovery**: 提供一個用戶友好的網站，允許開發人員搜尋和發現可在專案中使用的新套件。
2. **Package installation**: 使用 pip 指令可以輕鬆安裝軟體包，pip 指令是 Python 事實上的標準軟體包安裝程式。這使得開發人員可以快速安裝和使用第三方軟體包。
3. **Package versioning**: 維護所有軟體包版本的記錄，允許開發人員安裝和使用軟體包的特定版本。
4. **Package dependencies**: 允許包指定它們對其他包的依賴關係。這可確保開發人員在安裝特定軟體包時安裝所有必需的軟體包。
5. **Community support**: 由 Python 社群維護。它擁有一個充滿活力的貢獻者生態系統，幫助維護和更新軟體包。這意味著開發人員可以依靠社群為他們使用的軟體包提供支援和更新。

使用 PyPI 可以為開發人員提供對可合併到專案中的預先建置功能的訪問，從而節省開發人員的時間和精力，從而縮短開發時間並提供更強大的軟體應用程式。

## 套件的組成部分

Python 套件的主要元件是：

1. **Package 目錄**: 包含套件的模組，包括特殊的 `init.py` 檔案。

2. **init.py**: 導入套件時執行的特殊 Python 檔案。它可以包含初始化套件的程式碼以及套件內其他模組可存取的變數、函數和類別的定義。

3. **Module**: 包含可由其他模組執行或匯入的程式碼的 Python 檔案。一個套件可以包含一個或多個模組。

4. **Subpackages**: 一個套件可以包含一個或多個子套件，這些子套件本身就是包含自己的模組和子套件的套件。

5. **Setup 檔案**: 定義有關套件的元數據，例如其名稱、版本、作者和依賴項。 pip 等套件管理器使用此檔案來安裝套件及其相依性。

6. **README 檔案**: 提供有關套件的信息，例如其用途、安裝說明和使用範例。

7. **License 檔案**: 指定可以使用該套件的條款。

總體而言，Python 套件的組件協同工作，以模組化且可維護的方式組織和分發程式碼。

## 如何使用 PyPI

1. Install pip

若要安裝 pip，請開啟命令提示字元或終端機並輸入命令 `python -m Ensurepip --default-pip`。如果尚未安裝 pip，這將安裝 pip。

2. Search for packages

您可以使用 pip 指令 `pip search "package-name"` 在 PyPI 上搜尋套件。將 “package-name” 替換為您要搜尋的套件的名稱。

3. Install packages

若要從 PyPI 安裝軟體包，請使用 pip 指令 `pip install "package-name"`。將 “package-name” 替換為您要安裝的軟體包的名稱。

4. Use packages

安裝套件後，您可以透過匯入它在 Python 程式碼中使用它。例如，要使用 NumPy 套件，您可以將 `import numpy` 行新增至程式碼。

5. Upgrade packages

若要將軟體套件升級至 PyPI 上可用的最新版本，請使用 pip 命令 `pip install --upgrade "package-name"`。將 “package-name” 替換為您要升級的套件的名稱。

6. Uninstall packages

若要解除安裝不再需要的軟體包，請使用 pip 指令 `pip uninstall "package-name"`。將 “package-name” 替換為要卸載的套件的名稱。

## 如何建立套件並將其上傳到 PyPI

![](./create-first-python-package/creat_upload_package_pypi.avif)

### 前置準備

先使用 Python 虛擬環境(conda, venv) 來構建一個專案虛擬環境，環境啟動後安裝下列的工具:

```bash
pip install setuptools twine
```

### Step 1: 建立專案目錄

建立 Python 套件的第一步是建立一個新的專案目錄。這將包含該包所需的所有檔案和目錄。

若要建立新的專案目錄，請開啟終端機或命令提示字元並執行以下命令：

```bash
mkdir mypackage

cd mypackage
```

!!! info
    將 “mypackage” 替換為您的套件的名稱。

### Step 2: 建立套件目錄

下一步是在專案目錄中建立一個新目錄來保存套件代碼。它應該與您的套件具有相同的名稱。

若要建立新的套件目錄，請執行以下命令：

```bash
mkdir mypackage
```

!!! info
    將 “mypackage” 替換為您的套件的名稱。

### Step 3: 建立模組文件

在套件目錄中，建立一個新的 Python 檔案來保存套件代碼。該檔案應與您的套件同名，並帶有 `.py` 副檔名。

例如，如果您的套件名稱為 “mypackage”，請建立一個名為 “mypackage.py” 的檔案。

### Step 4: 編寫要封裝的程式碼

在剛剛建立的模組檔案中寫入套件代碼。它可以包含您想要在套件中包含的任何 Python 程式碼，例如函數、類別和變數。

### Step 5: 建立一個 setup.py 文件

要將 Python 套件發佈到 PyPI，您需要建立一個 `setup.py` 檔案。它包含有關套件的元數據，例如名稱、版本、作者和依賴項。

在專案目錄下新建一個名為 “setup.py” 的文件，並新增以下程式碼：

```python title="setup.py"
from setuptools import setup, find_packages

setup(
    name="mypackage",
    version="0.1.0",
    author="Your Name",
    author_email="your.email@example.com",
    description="A short description of your package",
    packages=find_packages(),
    classifiers=[
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
    ],
    python_requires=">=3.6",
)
```

### Step 6: 建置套件

若要建置套件，請在專案目錄中執行以下命令：

```bash
python setup.py sdist
```

這將在名為 `dist` 的新目錄中建立套件的來源發行版。

### Step 7: 將套件上傳到 PyPI

要將套件上傳到 PyPI，您首先需要在 PyPI 網站上建立帳戶。

完成後，在專案目錄中執行以下命令：

```bash
twine upload dist/*
```

這會將您的套件上傳到 PyPI。

### Step 8: 安裝套件

若要從 PyPI 安裝體套件，請執行以下命令：

```bash
pip install mypackage
```

!!! info
    將 “mypackage” 替換為您的套件的名稱。


## 從頭開始創建套件

### Step 1: 建立專案目錄

為您的專案建立一個新目錄並在 termianl 中切換到該目錄：

```bash
mkdir mathutils

cd mathutils
```

### Step 2: 建立套件目錄

下一步是在專案目錄中建立一個新目錄來保存套件代碼。它應該與您的套件具有相同的名稱。

```bash
mkdir mathutils
```

### Step 3: 建立模組文件

在套件目錄中建立兩個新的 Python 檔案：

- `operations.py`
- `geometry.py`

### Step 4: 編寫要封裝的程式碼

在 `operations.py`中，將一些基本數學運算定義為函數：

```python title="operations.py"
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b

def divide(a, b):
    return a / b
```

在 `geometry.py` 中，定義一些用來計算圓的面積和周長的函數：

```python title="geometry.py"
import math

def circle_area(radius):
    return math.pi * radius**2

def circle_circumference(radius):
    return 2 * math.pi * radius
```

### Step 5: 建立一個 setup.py 文件

它包含有關套件的元數據，例如名稱、版本、作者和依賴項。

在專案目錄下新建一個名為 `setup.py` 的文件，它包含有關套件的元數據，例如名稱、版本、作者和依賴項，並新增以下程式碼：

```python title="setup.py"
from setuptools import setup, find_packages

setup(
    name="mathutils",
    version="0.1.0",
    description="A simple Python package for math operations and geometry calculations",
    packages=find_packages(),
    classifiers=[
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
    ],
    python_requires=">=3.6",
)
```

### Step 6: 建置套件

若要建置套件，請在專案目錄中執行以下命令：

```bash
python setup.py sdist
```

這將在名為 `dist` 的新目錄中建立套件的來源發行版。

### Step 7: 將套件上傳到 PyPI

要將套件上傳到 PyPI，您首先需要在 PyPI 網站上建立帳戶。

完成後，在專案目錄中執行以下命令：

```bash
twine upload dist/*
```

這會將您的套件上傳到 PyPI。

### Step 8: 安裝套件

若要從 PyPI 安裝體套件，請執行以下命令：

```bash
pip install mathutils
```

## 結論

總之，創建 Python 套件並將其發佈到 PyPI 是任何 Python 開發人員的基本技能。透過與更廣泛的社群分享您的程式碼，您可以幫助其他人解決問題並在您的工作基礎上創建更強大的工具和應用程式。

在本文中，我們逐步探索了創建和發布 Python 套件的過程，從編寫程式碼、測試和建置套件到將其發佈到 PyPI 上。我們還討論了打包的最佳實踐，例如指定依賴項和確保與不同版本的 Python 的兼容性。

透過遵循這些實踐並投入時間正確打包和記錄程式碼，您可以使您的套件對其他人更易於存取和用戶友好。您還可以吸引其他開發人員加入您的項目，他們可以為您的程式碼做出貢獻、報告錯誤並幫助長期維護您的套件。

除了創建和發布包所需的技術技能之外，還需要考慮重要的軟技能。溝通、協作和對回饋的開放態度對於成功的套件開發和維護至關重要，並且可以幫助確保您的套件隨著時間的推移保持相關性和有用性。