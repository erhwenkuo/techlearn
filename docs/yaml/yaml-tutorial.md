# YAML 教程：5 分鐘上手

原文: https://www.educative.io/blog/yaml-tutorial

![](./assets/learn-yaml.webp)

YAML 是一種數據序列化語言，可讓您以緊湊易讀的格式存儲複雜數據。這對於 DevOps 和虛擬化很重要，因為它對於構建高效的數據管理系統和自動化至關重要。

雖然經常被開發人員忽視，但它是一個強大而簡單的工具，只需幾個小時的學習，就可以極大地改善你的工作前景。

今天，我們將通過動手教程幫助您快速學習 YAML，並探索如何在您的下一個數據驅動解決方案中使用它。

## 什麼是 YAML？

YAML 是一種數據序列化語言，用於以人類可讀的形式存儲信息。它最初代表 “Yet Another Markup Language”，但後來改為 “YAML Ain't Markup Language”，以區別於真正的標記語言。

它類似於 XML 和 JSON 文件，但即使在保持類似功能的同時使用更簡約的語法。 YAML 通常用於在 **基礎架構即代碼** (IaC) 程序中創建配置文件或在 DevOps 開發管道中管理容器。

最近，YAML 已被用於創建可以執行 YAML 文件中列出的一系列命令的自動化協議。這意味著您的系統可以更加獨立且響應迅速，而無需額外的開發人員關注。

隨著越來越多的公司採用 DevOps 和虛擬化，YAML 正迅速成為現代開發人員職位的必備技能。通過支持流行的技術，如使用 PyYAML 庫、Docker 或 Ansible 的 Python，YAML 也很容易與現有技術結合。

## YAML 的顯著特點

以下是 YAML 提供的一些顯著的功能。

### Multi-document 支持

您可以在一個 YAML 文件中包含多個 YAML 文檔，以使文件組織或數據解析更容易。

每個文檔之間的分隔使用用三個破折號 (`---`) 來做標記

```yaml
---
player: playerOne
action: attack (miss)
---
player: playerTwo
action: attack (hit)
---
```

### Built-in 註釋

YAML 允許您使用類似於 Python 註釋的井號 (`#`) 向文件添加 **註釋**。

```yaml
key: 
   #Here is a single-line comment 
   - value line 5
   #Here is a 
   #multi-line comment
   - value line 13
```

### 易讀語法

YAML 文件使用類似於 Python 的 **縮進系統** 來顯示程序的結構。您需要使用`space`而不是`tab`來創建縮進以避免混淆。

它還消除了 JSON 和 XML 文件中的大部分“noise”格式，例如引號、方括號和大括號。

這些格式規範提高了 YAML 文件的可讀性，超越了 XML 和 JSON。


```yaml title="YAML Example"
Imaro:
  author: Charles R. Saunders
  language: English
  publication-year: 1981
  pages: 224
```


```json title="JSON Example"
{
  "Imaro": {
    "author": "Charles R. Saunders",
    "language": "English",
    "publication-year": "1981",
    "pages": 224,
  }
}
```

請注意，上面兩個範例傳達了相同的信息；但是，在 YAML 文件中刪除了雙引號、逗號和方括號使其更易於一目了然。

### 隱式和顯式類型

YAML 通過自動檢測數據類型提供了輸入的多功能性，同時還支持顯式輸入選項。要將數據標記為特定類型，只需在值前包含 `!![typeName]`。

```yaml
# The value should be an int:
is-an-int: !!int 14.10
# Turn any value to a string:
is-a-str: !!str 67.43
# The next value should be a boolean:
is-a-bool: !!bool true
```

## YAML 語法

YAML 有一些構成大部分數據的基本概念。

### 鍵值對 Key-value pairs

通常，YAML 文件中的大多數內容都是 **鍵值對** 的一種形式，其中`鍵`代表鍵值對的`名稱`，`值`代表鍊接到該`名稱`的`數據`。**鍵值對** 是所有其他 YAML 結構的基礎。

### 標量和映射

標量表示單個存儲值。使用映射將標量分配給鍵名。您使用名稱、冒號和空格定義一個映射，然後是一個要保存的值。

YAML 支持常見類型，如整數和浮點數值，以及非數字類型 Boolean 和 String。

每個都可以用不同的方式表示，如十六進制、八進製或指數。還有一些特殊類型的數學概念，如無窮大、-無窮大和非數字 (NAN)

```yaml
integer: 25
hex: 0x12d4 #evaluates to 4820
octal: 023332 #evaluates to 9946

float: 25.0
exponent: 12.3015e+05 #evaluates to 1230150.0

boolean: Yes
string: "25"

infinity: .inf # evaluates to infinity
neginf: -.Inf #evaluates to negative infinity
not: .NAN #Not a Number
```

### 字符串 String

字符串是表示句子或短語的字符集合。您要麼使用 `|` 將每個字符串打印為新行或 `>` 將其打印為段落。

YAML 中的字符串不需要用雙引號。

```yaml
str: Hello World
data: |
   These
   Newlines
   Are broken up
data: >
   This text is
   wrapped and is a
   single paragraph
```

### 序列 Sequence

**序列** 是類似於 **列表** (list) 或 **數組** (array) 的數據結構，在同一個`鍵`下保存多個`值`。**序列** 可使用 `block` 或 `inline-flow` 樣式定義的。

`block` 樣式使用多行的 `-` 來建構文檔。與 `inline-flow` 風格相比，它更易於閱讀，但不太緊湊。

```yaml
--- 
# Shopping List Sequence in Block Style
shopping: 
- milk
- eggs
- juice
```

`inline-flow` 樣式允許您使用方括號內聯編寫 **序列**。 Flow 風格緊湊而且一目了然。

```yaml
--- 
# Shopping List Sequence in Flow Style
shopping: [milk, eggs, juice]
```

### 字典 Dictionaries

**字典** 是所有嵌套在同一子組下的 **鍵值對** 的集合。它們有助於將數據劃分為邏輯類別以供以後使用。

字典的定義類似於映射，您輸入字典名稱、冒號和空格，後面跟從著 1 個或多個 **縮進** 鍵值對。

```yaml
# An employee record
Employees: 
- dan:
    name: Dan D. Veloper
    job: Developer
    team: DevOps
- dora:
   name: Dora D. Veloper
   job: Project Manager
   team: Web Subscriptions
```


