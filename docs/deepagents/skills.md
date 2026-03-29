# Skills

> 學習如何透過 skills 擴展 deepagent 的功能

Skills 是可重複使用的 agnet 能力，提供專門的工作流程和領域知識。

您可以使用 [Agent Skills](https://agentskills.io/) 為您的 deep agent 賦予新的功能和專業知識。

Deep agent 的 skills 遵循 [Agent Skills specification](https://agentskills.io/specification)。

## What are skills

Skills 是一個資料夾目錄，其中每個資料夾包含一個或多個文件，這些文件包含代理可以使用的上下文資訊：

- 包含技能說明和元資料的 `SKILL.md` 文件
- 其他腳本（可選）
- 其他參考資訊，例如文件（可選）
- 其他資源，例如模板和其他資源（可選）

!!! info
    任何其他資產（腳本、文件、範本或其他資源）都必須在 `SKILL.md` 文件中引用，並提供有關該文件包含的內容以及如何使用它的信息，以便 agent 可以決定何時使用它們。

## How skills work

建立 deep agent 時，您可以傳入一個包含 skills 的目錄清單。Agent 啟動時，會讀取每個 `SKILL.md` 檔案的 **frontmatter** 部分。

當 agent 收到 prompt 時，它會檢查自身是否具備完成該 prompt 所需的技能。如果找到匹配的提示，它才會查看剩餘的技能文件。這種僅在需要時才查看技能資訊的模式稱為漸進式揭露 (progressive disclosure)。

## Example

你可能有一個 skills 資料夾，其中包含以特定方式使用文件網站的技能，以及搜尋 arXiv 預印本研究論文庫的另一個技能：

```bash
skills/
    ├── langgraph-docs
    │   └── SKILL.md
    └── arxiv_search
        ├── SKILL.md
        └── arxiv_search.py # code for searching arXiv
```

`SKILL.md` 檔案始終遵循相同的模式，從 **frontmatter** 中的元資料開始，然後是技能的說明。

以下範例展示了一項技能，該技能會在提示時提供如何提供相關語言圖文件的說明：

```markdown
---
name: langgraph-docs
description: Use this skill for requests related to LangGraph in order to fetch relevant documentation to provide accurate, up-to-date guidance.
---

# langgraph-docs

## Overview

This skill explains how to access LangGraph Python documentation to help answer questions and guide implementation.

## Instructions

### 1. Fetch the Documentation Index

Use the fetch_url tool to read the following URL:
https://docs.langchain.com/llms.txt

This provides a structured list of all available documentation with descriptions.

### 2. Select Relevant Documentation

Based on the question, identify 2-4 most relevant documentation URLs from the index. Prioritize:

- Specific how-to guides for implementation questions
- Core concept pages for understanding questions
- Tutorials for end-to-end examples
- Reference docs for API details

### 3. Fetch Selected Documentation

Use the fetch_url tool to read the selected documentation URLs.

### 4. Provide Accurate Guidance

After reading the documentation, complete the user's request.
```

更多技能範例，請參閱 [Deep Agent example skills](https://github.com/langchain-ai/deepagents/tree/main/libs/cli/examples/skills)。

!!! warning "Important"
    請參閱完整的 [Agent Skills Specification](https://agentskills.io/specification)，以了解撰寫技能文件時的限制和最佳實踐。尤其需要注意的是：

    1. 如果 `description` 字段長度超過 1024 個字符，則會被截斷為 1024 個字符。
    2. 在 Deep Agents 中，`SKILL.md` 檔案必須小於 10 MB。超出此限制的檔案在技能載入期間將被跳過。


### Full example

以下範例展示了一個使用所有可用 **frontmatter** 欄位的 `SKILL.md` 檔案：

```markdown
---
name: langgraph-docs
description: Use this skill for requests related to LangGraph in order to fetch relevant documentation to provide accurate, up-to-date guidance.
license: MIT
compatibility: Requires internet access for fetching documentation URLs
metadata:
  author: langchain
  version: "1.0"
allowed-tools: fetch_url
---

# langgraph-docs

## Overview

This skill explains how to access LangGraph Python documentation to help answer questions and guide implementation.

## Instructions

### 1. Fetch the documentation index

Use the fetch_url tool to read the following URL:
https://docs.langchain.com/llms.txt

This provides a structured list of all available documentation with descriptions.

### 2. Select relevant documentation

Based on the question, identify 2-4 most relevant documentation URLs from the index. Prioritize:

- Specific how-to guides for implementation questions
- Core concept pages for understanding questions
- Tutorials for end-to-end examples
- Reference docs for API details

### 3. Fetch selected documentation

Use the fetch_url tool to read the selected documentation URLs.

### 4. Provide accurate guidance

After reading the documentation, complete the user's request.
```

## Usage

建立 deep agent 時，請傳遞 skills 目錄：

=== "StateBackend"

    ```python
    from urllib.request import urlopen
    from deepagents import create_deep_agent
    from deepagents.backends.utils import create_file_data
    from langgraph.checkpoint.memory import MemorySaver

    checkpointer = MemorySaver()

    skill_url = "https://raw.githubusercontent.com/langchain-ai/deepagents/refs/heads/main/libs/cli/examples/skills/langgraph-docs/SKILL.md"

    with urlopen(skill_url) as response:
        skill_content = response.read().decode('utf-8')

    skills_files = {
        "/skills/langgraph-docs/SKILL.md": create_file_data(skill_content)
    }

    agent = create_deep_agent(
        skills=["./skills/"],
        checkpointer=checkpointer,
    )

    result = agent.invoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "What is langgraph?",
                }
            ],
            # Seed the default StateBackend's in-state filesystem (virtual paths must start with "/").
            "files": skills_files
        },
        config={"configurable": {"thread_id": "12345"}},
    )
    ```

=== "StoreBackend"

    ```python
    from urllib.request import urlopen
    from deepagents import create_deep_agent
    from deepagents.backends import StoreBackend
    from deepagents.backends.utils import create_file_data
    from langgraph.store.memory import InMemoryStore


    store = InMemoryStore()

    skill_url = "https://raw.githubusercontent.com/langchain-ai/deepagents/refs/heads/main/libs/cli/examples/skills/langgraph-docs/SKILL.md"
    with urlopen(skill_url) as response:
        skill_content = response.read().decode('utf-8')

    store.put(
        namespace=("filesystem",),
        key="/skills/langgraph-docs/SKILL.md",
        value=create_file_data(skill_content)
    )

    agent = create_deep_agent(
        backend=(lambda rt: StoreBackend(rt)),
        store=store,
        skills=["/skills/"]
    )

    result = agent.invoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "What is langgraph?",
                }
            ]
        },
        config={"configurable": {"thread_id": "12345"}},
    )
    ```

=== "FilesystemBackend"

    ```python
    from deepagents import create_deep_agent
    from langgraph.checkpoint.memory import MemorySaver
    from deepagents.backends.filesystem import FilesystemBackend

    # Checkpointer is REQUIRED for human-in-the-loop
    checkpointer = MemorySaver()

    agent = create_deep_agent(
        backend=FilesystemBackend(root_dir="/Users/user/{project}"),
        skills=["/Users/user/{project}/skills/"],
        interrupt_on={
            "write_file": True,  # Default: approve, edit, reject
            "read_file": False,  # No interrupts needed
            "edit_file": True    # Default: approve, edit, reject
        },
        checkpointer=checkpointer,  # Required!
    )

    result = agent.invoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "What is langgraph?",
                }
            ]
        },
        config={"configurable": {"thread_id": "12345"}},
    )
    ```


**skills** `list[str]`

- skill 來源路徑清單。
- 路徑必須使用正斜線(`/`)指定，並且相對於後端根目錄。
    - 使用 `StateBackend`（預設）時，請使用 `invoke(files={...})` 提供 skill 檔。請使用 `deepagents.backends.utils` 中的 `create_file_data()` 函數格式化檔案內容；不支援原始字串。
    - 使用 `FilesystemBackend` 時，skill 是從相對於後端根目錄的磁碟載入的。

對於同名的 skill 技能，後發布的資料會覆蓋先前的資料（後發布的資料優先）。

## Source precedence

當多個 skill source 包含同名技能時，技能清單中後面列出的技能來源優先（後來新增的技能生效）。這樣就可以疊加來自不同來源的技能。

```python
# If both sources contain a skill named "web-search",
# the one from "/skills/project/" wins (loaded last).
agent = create_deep_agent(
    skills=["/skills/user/", "/skills/project/"],
    ...
)
```

## Skills for subagents

使用 [subagents](https://docs.langchain.com/oss/python/deepagents/subagents) 時，您可以配置每種類型的 agent 可以存取哪些技能：

- **General-purpose subagent**: 當您將 skills 傳遞給 `create_deep_agent` 時，它會自動繼承主代理的技能。無需額外配置。
- **Custom subagents**: 不要繼承 main agent 的技能。在每個 subagent 定義中新增一個 skills parameter，並指定該 subagent 的技能來源路徑。

Skill state 完全隔離：main agent 的技能對子代理不可見，subagent 的技能對主代理也不可見。

```python
from deepagents import create_deep_agent

research_subagent = {
    "name": "researcher",
    "description": "Research assistant with specialized skills",
    "system_prompt": "You are a researcher.",
    "tools": [web_search],
    "skills": ["/skills/research/", "/skills/web-search/"],  # Subagent-specific skills
}

agent = create_deep_agent(
    model="claude-sonnet-4-5-20250929",
    skills=["/skills/main/"],  # Main agent and GP subagent get these
    subagents=[research_subagent],  # Researcher gets only its own skills
)
```

有關 subagent 配置和 skills 繼承的更多信息，請參閱 [Subagents](https://docs.langchain.com/oss/python/deepagents/subagents)。

## What the agent sees

配置 skills 後，會在 agnet 系統提示中插入一個 Skills System 部分。Agent 會使用此資訊執行以下三個步驟：

1. **Match** — 當使用者發出 prompt 時，agent 會檢查是否有任何 skill 的描述與任務相符。
2. **Read** — 如果某項 skill 適用，agent 將使用其技能清單中顯示的路徑讀取完整的 `SKILL.md` 檔案。
3. **Execute** — Agent 依照 skill 的指示執行操作，並根據需要存取任何支援文件（腳本、範本、參考文件）。

!!! tip
    請在 `SKILL.md` 檔案的 **frontmatter** 資料中編寫清晰、具體的 skill 描述。Agent 僅根據描述來決定是否使用該 skill — 詳細的描述有助於更精準地匹配 skill。

## Skills vs. memory

Skills 和 memory（`AGENTS.md` 檔案）用途不同：

|   | Skills | Memory |
|---|--------|--------|
|**目的**|透過漸進式揭露發現的按需功能|持久上下文始終在啟動時加載|
|**載入**|僅當代理確定相關時才讀取|始終注入到系統提示(system prompt)字元中|
|**格式**|`SKILL.md` 檔案位於已命名的目錄中|`AGENTS.md` 文件|
|**分層**|User → project (last wins)|User → project (combined)|
|**使用時機**|指令針對特定任務，且可能內容較長。|Ｃontext 始終很重要（專案慣例、偏好）|

## When to use skills and tools

以下是一些使用 tools 和 skills 的一般準則：

- 當上下文資訊豐富時，使用 skills 可以減少系統提示中的 tokens 數量。
- 使用 skills 將能力捆綁成更大的操作能力，並提供除單一工具描述之外的更多上下文資訊。
- 如果 agent 沒有檔案系統存取權限，則使用 tools。
