# What are skills?

> Agent Skills 是一種輕量級的開放格式，可透過專業知識和工作流程擴展 AI agent 的能力。

Skill 的核心是一個包含 `SKILL.md` 檔案的資料夾。該檔案包含元資料（至少包括 `name` 和 `description`）以及指示 agent 如何執行特定任務的指令。Skill 還可以包含腳本、範本和參考資料。

```bash
my-skill/
├── SKILL.md          # Required: instructions + metadata
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
└── assets/           # Optional: templates, resources
```

## How skills work

Skill 運用漸進式揭露(progressive disclosure)來有效管理背景資訊：

1. **Discovery**: 啟動時，agent 只會載入每項可用技能的名稱和描述，僅足以知道何時可能相關。
2. **Activation**: 當任務與技能描述相符時，agent 會將完整的 `SKILL.md` 指令解讀到上下文中。
3. **Execution**: Agent 會依照指令執行，並可依需求載入引用的檔案或執行捆綁程式碼。

這種方法既能保證 agent 快速回應，又能讓他們根據需要獲取更多上下文資訊。


## The `SKILL.md` file

每個 skill 都以一個包含 YAML 前元資料和 Markdown 指令的 `SKILL.md` 檔案開始：

```md
---
name: pdf-processing
description: Extract text and tables from PDF files, fill forms, merge documents.
---

# PDF Processing

## When to use this skill
Use this skill when the user needs to work with PDF files...

## How to extract text
1. Use pdfplumber for text extraction...

## How to fill forms
...
```

SKILL.md 檔案頂部必須包含以下前元資料：

- `name`: 簡短的標識符
- `description`: 何時使用這項技能

Markdown 正文包含實際的指令，對結構或內容沒有任何具體限制。

這種簡單的格式具有一些主要優點：

- **Self-documenting**: Skill 作者或使用者可以閱讀 `SKILL.md` 文件，了解其功能，從而方便技能的審查和改進。
- **Extensible**: Skill 的複雜程度可以從簡單的文字指令到可執行程式碼、資源和模板不等。
- **Portable**: Skill 只是文件，因此很容易編輯、版本控制和共享。

## Next steps

- [查看規格說明](https://agentskills.io/specification)以了解完整格式。
- 為您的 agent [增加技能支持](https://agentskills.io/integrate-skills)，以建立相容的的 agnet 實現。
- 請查看 GitHub 上的[範例技能](https://github.com/anthropics/skills)。
- [閱讀 Skill 寫作最佳實踐](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)，提升寫作技巧。
- 使用[參考庫](https://github.com/agentskills/agentskills/tree/main/skills-ref)驗證技能並產生提示 XML。