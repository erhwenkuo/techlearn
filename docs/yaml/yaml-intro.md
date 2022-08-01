# YAML 入門教程

![](./assets/learn-yaml.webp)

YAML 是 "YAML Ain't a Markup Language"（YAML 不是一種標記語言）的縮寫。

YAML 的語法和其他高級語言類似，並且可以簡單表達清單、散列表，標量等數據形態。它使用空白符號縮進和大量依賴外觀的特色，特別適合用來表達或編輯數據結構、各種配置文件、傾印調試內容、文件大綱（例如：許多電子郵件標題格式和 YAML 非常接近）。

YAML 的配置文件後綴為 `.yml`，如：runoob.yml 。

## 基本語法

- 大小寫敏感
- 使用縮進表示層級關係
- 縮進不允許使用tab，只允許空格
- 縮進的空格數不重要，只要相同層級的元素左對齊即可
- '#'表示註釋

## 數據類型

YAML 支持以下幾種數據類型：

- 物件/對象：鍵值對的集合，又稱為映射（mapping）/ 哈希（hashes） / 字典（dictionary）
- 陣列/數組：一組按次序排列的值，又稱為序列（sequence） / 列表（list）
- 純量（scalars）：單個的、不可再分的值

### 物件/對象

對象鍵值對使用 **冒號** 結構表示 `key: value`，==冒號後面要加一個空格==。

也可以使用 `key:{key1: value1, key2: value2, ...}`。

還可以使用縮進表示層級關係；

```yaml
key: 
    child-key: value
    child-key2: value2
```

較為複雜的對象格式，可以使用問號加一個空格代表一個複雜的 key，配合一個冒號加一個空格代表一個 value：

```yaml
?  
    - complexkey1
    - complexkey2
:
    - complexvalue1
    - complexvalue2
```

意思即對象的屬性是一個數組 [complexkey1,complexkey2]，對應的值也是一個數組 [complexvalue1,complexvalue2]

### 陣列/數組

以 `-` 開頭的行表示構成一個數組：

```yaml
key：
- A
- B
- C
```

YAML 支持多維數組，可以使用 inline 格式來表示：

```yaml
key: [A, B, C, ...]
```

數據結構的子成員是一個數組，則可以在該項下面縮進一個空格。

```yaml
-
 - A
 - B
 - C
```

一個相對複雜的例子：

```yaml
companies:
    -
        id: 1
        name: company1
        price: 200W
    -
        id: 2
        name: company2
        price: 500W
```

意思是 companies 屬性是一個數組，每一個數組元素又是由 id、name、price 三個屬性構成。

數組也可以使用流式(flow)的方式表示：

```yaml
companies: [{id: 1,name: company1,price: 200W},{id: 2,name: company2,price: 500W}]
```

## 複合結構

數組和對象可以構成複合結構，例如：

```yaml
languages:
  - Ruby
  - Perl
  - Python 
websites:
  YAML: yaml.org 
  Ruby: ruby-lang.org 
  Python: python.org 
  Perl: use.perl.org
```

轉換為 json 為：

```json
{ 
  languages: [ 'Ruby', 'Perl', 'Python'],
  websites: {
    YAML: 'yaml.org',
    Ruby: 'ruby-lang.org',
    Python: 'python.org',
    Perl: 'use.perl.org' 
  } 
}
```

## 純量

純量是最基本的，不可再分的值，包括：

- 字符串
- 布爾值
- 整數
- 浮點數
- Null
- 時間
- 日期

使用一個例子來快速了解純量的基本使用：

```yaml
boolean: 
    - TRUE  #true,True都可以
    - FALSE  #false，False都可以
float:
    - 3.14
    - 6.8523015e+5  #可以使用科學計數法
int:
    - 123
    - 0b1010_0111_0100_1010_1110    #二進製表示
null:
    nodeName: 'node'
    parent: ~  #使用~表示null
string:
    - 哈哈
    - 'Hello world'  #可以使用雙引號或者單引號包裹特殊字符
    - newline
      newline2    #字符串可以拆成多行，每一行會被轉化成一個空格
date:
    - 2018-02-17    #日期必須使用ISO 8601格式，即yyyy-MM-dd
datetime: 
    - 2018-02-17T15:02:31+08:00    #時間使用ISO 8601格式，時間和日期之間使用T連接，最後使用+代表時區
```

## 引用

`&` 錨點和 `*` 別名，可以用來引用:

```yaml
defaults: &defaults
  adapter:  postgres
  host:     localhost

development:
  database: myapp_development
  <<: *defaults

test:
  database: myapp_test
  <<: *defaults
```

相當於:

```yaml
defaults:
  adapter:  postgres
  host:     localhost

development:
  database: myapp_development
  adapter:  postgres
  host:     localhost

test:
  database: myapp_test
  adapter:  postgres
  host:     localhost
```

`&` 用來建立錨點（defaults），`<<` 表示合併到當前數據，`*` 用來引用錨點。

下面是另一個例子:

```yaml
- &showell Steve 
- Clark 
- Brian 
- Oren 
- *showell 
```

轉為 Json 結構如下:

```json
[ 'Steve', 'Clark', 'Brian', 'Oren', 'Steve' ]
```
