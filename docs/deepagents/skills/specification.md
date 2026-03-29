# Specification

> Agent Skills 的完整格式規格。

本文檔定義了 Agent Skills 格式。

## Directory structure

Skill 是一個目錄，其中至少包含一個 `SKILL.md` 檔案：

```bash
skill-name/
└── SKILL.md          # Required
```

!!! tip
    您也可以選擇新增其他目錄，例如 `scripts/`, `references/` 和 `assets/`，以支援 skill。

## `SKILL.md` format

`SKILL.md` 檔案必須包含 YAML 前元數據，後面接著 Markdown 內容。

### Frontmatter (required)

```bash
---
name: skill-name
description: A description of what this skill does and when to use it.
---
```

可增加一些 optional 的欄位：

```bash
---
name: pdf-processing
description: Extract text and tables from PDF files, fill forms, merge documents.
license: Apache-2.0
metadata:
  author: example-org
  version: "1.0"
---
```

|Field	|Required	|Constraints|
|-------|-----------|-----------|
|`name`	|Yes	|最多64個字元。限小寫字母、數字和連字號。不得以連字符開頭或結尾。|
|`description`	|Yes	|最多 1024 個字元。非空。描述技能的效果以及何時使用。|
|`license`	|No	|許可證名稱或捆綁許可證文件的引用。|
|`compatibility`	|No	|最多 500 個字元。說明環境需求（目標產品、系統軟體包、網路存取等）。|
|`metadata`	|No	|任意鍵值映射，用於新增元資料。|
|`allowed-tools`|No	|Skill 可使用的預先核准工具列表，以空格分隔。（實驗性功能）|

#### `name` field

必定要有的 `name` 欄位：

- 必須為 1-64 個字符
- 只能包含 Unicode 小寫字母數字字元和連字符（a-z 和 -）。
- 不得以“-”開頭或結尾
- 不能包含連續的連字符（--）
- 必須與父目錄名稱相符

有效範例：

```bash
name: pdf-processing
```

```bash
name: data-analysis
```

```bash
name: code-review
```

無效範例：

```bash
name: PDF-Processing  # uppercase not allowed
```

```bash
name: -pdf  # cannot start with hyphen
```

```bash
name: pdf--processing  # consecutive hyphens not allowed
```

#### `description` field

必定要有的 `description` 欄位：

- 長度必須為 1-1024 個字符
- 應描述技能的功能以及何時使用
- 應包含有助於 agent 識別相關任務的特定關鍵字

好的範例：

```bash
description: Extracts text and tables from PDF files, fills PDF forms, and merges multiple PDFs. Use when working with PDF documents or when the user mentions PDFs, forms, or document extraction.
```

不好的範例：

```bash
description: Helps with PDFs.
```

#### `license` field

可選的 `license` 欄位：

- 指定應用於該技能的許可證
- 我們建議盡量簡短（可以是許可證名稱，也可以是捆綁許可證文件的名稱）。

範例：

```bash
license: Proprietary. LICENSE.txt has complete terms
```

#### `compatibility` field

可選的 `compatibility` 欄位：

- 如果提供，字數必須為 1-500 個字元。
- 僅當您的技能有特定環境要求時才應包含此項。
- 可以說明目標產品、所需的系統軟體包、網路存取需求等。

範例：

```bash
compatibility: Designed for Claude Code (or similar products)
```

```bash
compatibility: Requires git, docker, jq, and access to the internet
```

!!! info
    大多數 skills 不需要相容性欄位。

#### `metadata` field

可選的 `metadata` 欄位：

- 一個從字串鍵到字串值的映射表
- 客戶端可以使用此映射表儲存代理技能規格中未定義的其他屬性。
- 我們建議您使用盡可能唯一的鍵名，以避免意外衝突。

範例：

```bash
metadata:
  author: example-org
  version: "1.0"
```

#### `allowed-tools` field

可選的 `allowed-tools` 欄位：

- 預先核准運行的工具清單（以空格分隔）
- 實驗性功能。不同代理實現對此欄位的支援可能有所不同。

範例：

```bash
allowed-tools: Bash(git:*) Bash(jq:*) Read
```

### Body content

前元資料之後的 Markdown 正文包含技能說明。格式沒有限制。只需編寫有助於 agent 有效完成任務的內容即可。

建議的內容：

- Step-by-step 導引
- 輸入輸出範例
- 常見邊界狀況(edge cases)

請注意，agent 會在決定啟動某個 skill 時載入整個 `SKILL.md` 檔案。建議將較長的 `SKILL.md` 內容拆分成多個引用檔案。

## Optional directories

### `scripts/`

包含 agent 可以運行的可執行程式碼。腳本應：

- 獨立完整或清晰記錄依賴關係
- 包含有用的錯誤訊息
- 妥善處理特殊狀況

支援的語言取決於 agent 的具體實作。常見的語言包括 Python, Bash 和 JavaScript。

### `references/`

包含 agent 需要時可查閱的其他文件：

- `REFERENCE.md` - 詳細技術參考
- `FORMS.md` - 表單範本或結構化資料格式
- Domain-specific 檔案 (`finance.md`, `legal.md`, etc.)

保持各個參考文件(reference files)的簡潔性。Agent 會按需載入這些文件，因此文件越小，對上下文資訊的依賴就越少。

### `assets/`

包含靜態資源：

- 範本（文件範本、設定範本）
- 圖像（圖表、範例）
- 資料檔案（查找表、模式）

## Progressive disclosure

Skills 應結構化，以便有效利用 context：

1. **Metadata** (~100 tokens): 所有 skills 的名稱和描述欄位都會在啟動時載入。
2. **Instructions** (< 5000 tokens recommended): Skills 啟動時，完整的 `SKILL.md` 檔案內容將會載入。
3. **Resources** (as needed): 檔案 (e.g. 那些歸納在 `scripts/`, `references/`, 或 `assets/`) 僅在需要時加載

主文檔 `SKILL.md` 應控制在 500 行以內。詳細的參考資料應移至單獨的文件中。

## File references

在技​ skill 中引用其他檔案時，請使用相對於技能根目錄的相對路徑：

```bash
See [the reference guide](references/REFERENCE.md) for details.

Run the extraction script:
scripts/extract.py
```

文件引用應保持在 `SKILL.md` 的一級深度。避免使用深度嵌套的引用鏈。

## Validation

使用 [skills-ref 參考庫](https://github.com/agentskills/agentskills/tree/main/skills-ref)來驗證您的 skill：

```bash
skills-ref validate ./my-skill
```

這會檢查您的 SKILL.md 前元資料是否有效並符合所有命名約定。

