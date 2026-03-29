# Integrate skills into your agent

> 如何為您的 agent 或工具添加 Agent Skills 的支援。

本指南說明如何為 AI Agent 或 development tool 添加 skills 支援。

## Integration approaches

整合 skills 的兩種主要方法是：

- **Filesystem-based agents**: 基於檔案系統的 Agent 程式在電腦環境（bash/unix）中運行，是功能最強大的選擇。當模型發出類似 `cat /path/to/my-skill/SKILL.md` 的 `shell` 指令時，技能就會被啟動。捆綁的資源也是透過 `shell` 指令存取的。
- **Tool-based agents**: 基於 tool-based 的 agent 無需專用電腦環境即可運作。它們透過實現各種工具，使模型能夠觸發技能並存取捆綁的資源。具體的工具實現方式由開發者決定。

## Overview

具備 skills-compatible agent 需要：

1. **Discover**: 在已配置的目錄中發現技能
2. **Load metadata**: 啟動時載入元資料（名稱和描述）
3. **Match**: 將使用者任務與相關技能相匹配
4. **Activate**: 透過載入完整 instruction 來啟動技能
5. **Execute**: 根據需要執行腳本並存取資源

## Skill discovery

技能資料夾包含一個 `SKILL.md` 檔案。您的 agent 應掃描已配置的目錄以查找有效的 skills。

## Loading metadata

啟動時，僅解析每個 SKILL.md 檔案的 frontmatter。這樣可以保持初始上下文使用量較低。

### Parsing frontmatter

```js
function parseMetadata(skillPath):
    content = readFile(skillPath + "/SKILL.md")
    frontmatter = extractYAMLFrontmatter(content)

    return {
        name: frontmatter.name,
        description: frontmatter.description,
        path: skillPath
    }
```

### Injecting into context

在系統提示中包含 skill metadata，以便模型了解有哪些技能可用。

請遵循平台關於系統提示更新的指南。例如，對於 Claude 模型，建議使用 XML 格式：

```xml
<available_skills>
  <skill>
    <name>pdf-processing</name>
    <description>Extracts text and tables from PDF files, fills forms, merges documents.</description>
    <location>/path/to/skills/pdf-processing/SKILL.md</location>
  </skill>
  <skill>
    <name>data-analysis</name>
    <description>Analyzes datasets, generates charts, and creates summary reports.</description>
    <location>/path/to/skills/data-analysis/SKILL.md</location>
  </skill>
</available_skills>
```

對於 **filesystem-based agents**，請在 `location` 欄位中包含 `SKILL.md` 檔案的絕對路徑。對於 **tool-based agents**，可以省略 location 欄位。

保持元數據簡潔。每個技能應在上下文中添加大約 50-100 個 tokens。

## Security considerations

腳本執行會帶來安全風險。請考慮以下因素：

- **Sandboxing**: 在隔離環境中執行腳本
- **Allowlisting**: 僅執行來自可信任 skills 的腳本
- **Confirmation**: 在執行可能危險的操作之前，請先徵求使用者意見。
- **Logging**: 記錄所有腳本執行情況以進行審計

## Reference implementation

[skills-ref](https://github.com/agentskills/agentskills/tree/main/skills-ref) 函式庫提供 Python 實用程式和 CLI，用於處理 skills。

例如：

**Validate a skill directory**:

```bash
skills-ref validate <path>
```

**Generate <available_skills> XML for agent prompts**:

```bash
skills-ref to-prompt <path>...
```

使用[skills-ref](https://github.com/agentskills/agentskills/tree/main/skills-ref) 函式庫源代碼作為參考實作。

