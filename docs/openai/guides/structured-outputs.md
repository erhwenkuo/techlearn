# Structured Outputs

> 確保 LLM 的回應符合指定的 JSON schema。

[原文](https://platform.openai.com/docs/guides/structured-outputs)

## Introduction

JSON 是世界上應用程式交換資料最廣泛的格式之一。

結構化輸出(Structured Outputs)功能可確保 LLM 模型產生符合您提供的 JSON schema的回應，因此您無需擔心模型會遺漏必要的 key 或產生無效的 enum 值。

結構化輸出的一些好處包括：

1. **Reliable type-safety**: 無需驗證或重試格式不正確的回應
2. **Explicit refusals**: 現在可以透過程式檢測基於模型回應的安全性
3. **Simpler prompting**: 無需使用強烈的提示來實現格​​式的一致性

除了在 REST API 中支援 JSON Schema 之外，OpenAI 的 Python 和 JavaScript SDK 還支援分別使用 `[Pydantic](https://docs.pydantic.dev/latest/)` 和 `[Zod](https://zod.dev/)` 輕鬆定義物件 Schema。下文將示範如何從符合程式碼中定義的 Schema 的非結構化文字中提取資訊。

**獲得結構化的回應(Getting a structured response)**:

=== "python"

    ```python
    from openai import OpenAI
    from pydantic import BaseModel

    client = OpenAI()

    class CalendarEvent(BaseModel):
        name: str
        date: str
        participants: list[str]

    response = client.responses.parse(
        model="gpt-4o-2024-08-06",
        input=[
            {"role": "system", "content": "Extract the event information."},
            {
                "role": "user",
                "content": "Alice and Bob are going to a science fair on Friday.",
            },
        ],
        text_format=CalendarEvent,
    )

    event = response.output_parsed
    ```

=== "javascript"

    ```javascript
    import OpenAI from "openai";
    import { zodTextFormat } from "openai/helpers/zod";
    import { z } from "zod";

    const openai = new OpenAI();

    const CalendarEvent = z.object({
    name: z.string(),
    date: z.string(),
    participants: z.array(z.string()),
    });

    const response = await openai.responses.parse({
    model: "gpt-4o-2024-08-06",
    input: [
        { role: "system", content: "Extract the event information." },
        {
        role: "user",
        content: "Alice and Bob are going to a science fair on Friday.",
        },
    ],
    text: {
        format: zodTextFormat(CalendarEvent, "event"),
    },
    });

    const event = response.output_parsed;
    ```

### Supported models

從 GPT-4o 開始，OpenAI最新的大型語言模型已支援結構化輸出。 GPT-4-turbo 及更早版本的模型可能需要使用 JSON mode 來定義。

### Function calling vs text.format

OpenAI API 中有兩種結構化輸出(Structured Outputs)的手法：

1. 使用 function calling / tool calling 的時候
2. 使用 `json_schema` 定義回應格式的時候

當您需要 LLM 去連接到你外部的應用程式或系統時，function calling / tool calling 很有用。

例如，您可以讓 LLM 模型存取查詢資料庫的功能，以便建立可以幫助使用者下訂單的 AI 助手，或是可以與 UI 互動的功能。

相反，當您想要指示 LLM 模型回應使用者詢問時使用 structure schema，而不是模型呼叫工具時，透過 `response_format` 進行結構化輸出會更為合適。

例如，如果您正在建立數學小助手應用程序，您可能希望數學小助手使用特定的 JSON schema 來回應您的用戶，以便您可以產生以不同方式顯示模型輸出的不同部分的 UI。

簡單地說：

- 如果您將模型連接到系統中的工具、功能、資料等，那麼您應該使用 function calling / tool calling
- 如果您希望在模型響應用戶時建立其輸出，那麼您應該使用結構化的 `text.format`

!!! info
    本指南的其餘部分將重點介紹 OpenAI Responses API 中的 non-function calling 的用例。如需詳細了解如何將結構化輸出與 function calling 呼叫結合使用，請參閱 [Function Calling 指南](https://platform.openai.com/docs/guides/function-calling#function-calling-with-structured-outputs)。

### Structured Outputs vs JSON mode

Structured Outputs 是 JSON mode 的演進。雖然兩者都能確保產生有效的 JSON，但只有結構化輸出才能確保符合定義的 schema。Responses API, Chat Completions API, Assistants API, Fine-tuning API 和 Batch API 均支援結構化輸出和 JSON schema。

OpenAI 建議盡可能使用 Structured Outputs 而不是 JSON mode。

但是，具有 `response_format: {type: "json_schema", ...}` 的結構化輸出僅支援 `gpt-4o-mini`, `gpt-4o-mini-2024-07-18` 和 `gpt-4o-2024-08-06` 模型快照及更高版本。

| Feature                | Structured Outputs | JSON Mode |
|------------------------|--------------------|-----------|
| 輸出 valid JSON     | Yes                | Yes       |
| 遵守定義的 schema      | Yes (see supported schemas)  | No         |
| 相容 models | `gpt-4o-mini`, `gpt-4o-2024-08-06`, 與之後的模型 | `gpt-3.5-turbo`, `gpt-4-*` 與 `gpt-4o-*` 模型    |
| 如何啟用 | `text: { format: { type: "json_schema", "strict": true, "schema": ... } }` | `text: { format: { type: "json_object" } }` |

## Examples

### Chain of thought

您可以要求模型以結構化、逐步的方式輸出答案，以引導使用者完成解決方案。

**Structured Outputs for chain-of-thought math tutoring**

=== "curl" hl_lines="16-41"

    ```bash title="curl"
    curl https://api.openai.com/v1/responses \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "model": "gpt-4o-2024-08-06",
        "input": [
        {
            "role": "system",
            "content": "You are a helpful math tutor. Guide the user through the solution step by step."
        },
        {
            "role": "user",
            "content": "how can I solve 8x + 7 = -23"
        }
        ],
        "text": {
        "format": {
            "type": "json_schema",
            "name": "math_reasoning",
            "schema": {
            "type": "object",
            "properties": {
                "steps": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                    "explanation": { "type": "string" },
                    "output": { "type": "string" }
                    },
                    "required": ["explanation", "output"],
                    "additionalProperties": false
                }
                },
                "final_answer": { "type": "string" }
            },
            "required": ["steps", "final_answer"],
            "additionalProperties": false
            },
            "strict": true
        }
        }
    }'
    ```

=== "python" hl_lines="23 26"

    ```python
    from openai import OpenAI
    from pydantic import BaseModel

    client = OpenAI()

    class Step(BaseModel):
        explanation: str
        output: str

    class MathReasoning(BaseModel):
        steps: list[Step]
        final_answer: str

    response = client.responses.parse(
        model="gpt-4o-2024-08-06",
        input=[
            {
                "role": "system",
                "content": "You are a helpful math tutor. Guide the user through the solution step by step.",
            },
            {"role": "user", "content": "how can I solve 8x + 7 = -23"},
        ],
        text_format=MathReasoning,
    )

    math_reasoning = response.output_parsed
    ```

=== "javascript" hl_lines="27-29 32"

    ```javascript
    import OpenAI from "openai";
    import { zodTextFormat } from "openai/helpers/zod";
    import { z } from "zod";

    const openai = new OpenAI();

    const Step = z.object({
    explanation: z.string(),
    output: z.string(),
    });

    const MathReasoning = z.object({
    steps: z.array(Step),
    final_answer: z.string(),
    });

    const response = await openai.responses.parse({
    model: "gpt-4o-2024-08-06",
    input: [
        {
        role: "system",
        content:
            "You are a helpful math tutor. Guide the user through the solution step by step.",
        },
        { role: "user", content: "how can I solve 8x + 7 = -23" },
    ],
    text: {
        format: zodTextFormat(MathReasoning, "math_reasoning"),
    },
    });

    const math_reasoning = response.output_parsed;
    ```

#### Example response

```bash
{
  "steps": [
    {
      "explanation": "Start with the equation 8x + 7 = -23.",
      "output": "8x + 7 = -23"
    },
    {
      "explanation": "Subtract 7 from both sides to isolate the term with the variable.",
      "output": "8x = -23 - 7"
    },
    {
      "explanation": "Simplify the right side of the equation.",
      "output": "8x = -30"
    },
    {
      "explanation": "Divide both sides by 8 to solve for x.",
      "output": "x = -30 / 8"
    },
    {
      "explanation": "Simplify the fraction.",
      "output": "x = -15 / 4"
    }
  ],
  "final_answer": "x = -15 / 4"
}
```

### Structured data extraction

您可以定義結構化欄位來從非結構化輸入資料（例如研究論文）中提取。

**Extracting data from research papers using Structured Outputs**

=== "curl"

    ```bash
    curl https://api.openai.com/v1/responses \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "model": "gpt-4o-2024-08-06",
        "input": [
        {
            "role": "system",
            "content": "You are an expert at structured data extraction. You will be given unstructured text from a research paper and should convert it into the given structure."
        },
        {
            "role": "user",
            "content": "..."
        }
        ],
        "text": {
        "format": {
            "type": "json_schema",
            "name": "research_paper_extraction",
            "schema": {
            "type": "object",
            "properties": {
                "title": { "type": "string" },
                "authors": {
                "type": "array",
                "items": { "type": "string" }
                },
                "abstract": { "type": "string" },
                "keywords": {
                "type": "array",
                "items": { "type": "string" }
                }
            },
            "required": ["title", "authors", "abstract", "keywords"],
            "additionalProperties": false
            },
            "strict": true
        }
        }
    }'
    ```

=== "python"

    ```python
    from openai import OpenAI
    from pydantic import BaseModel

    client = OpenAI()

    class ResearchPaperExtraction(BaseModel):
        title: str
        authors: list[str]
        abstract: str
        keywords: list[str]

    response = client.responses.parse(
        model="gpt-4o-2024-08-06",
        input=[
            {
                "role": "system",
                "content": "You are an expert at structured data extraction. You will be given unstructured text from a research paper and should convert it into the given structure.",
            },
            {"role": "user", "content": "..."},
        ],
        text_format=ResearchPaperExtraction,
    )

    research_paper = response.output_parsed
    ```

=== "javascript"

    ```javascript
    import OpenAI from "openai";
    import { zodTextFormat } from "openai/helpers/zod";
    import { z } from "zod";

    const openai = new OpenAI();

    const ResearchPaperExtraction = z.object({
    title: z.string(),
    authors: z.array(z.string()),
    abstract: z.string(),
    keywords: z.array(z.string()),
    });

    const response = await openai.responses.parse({
    model: "gpt-4o-2024-08-06",
    input: [
        {
        role: "system",
        content:
            "You are an expert at structured data extraction. You will be given unstructured text from a research paper and should convert it into the given structure.",
        },
        { role: "user", content: "..." },
    ],
    text: {
        format: zodTextFormat(ResearchPaperExtraction, "research_paper_extraction"),
    },
    });

    const research_paper = response.output_parsed;
    ```

#### Example response

```bash
{
  "title": "Application of Quantum Algorithms in Interstellar Navigation: A New Frontier",
  "authors": [
    "Dr. Stella Voyager",
    "Dr. Nova Star",
    "Dr. Lyra Hunter"
  ],
  "abstract": "This paper investigates the utilization of quantum algorithms to improve interstellar navigation systems. By leveraging quantum superposition and entanglement, our proposed navigation system can calculate optimal travel paths through space-time anomalies more efficiently than classical methods. Experimental simulations suggest a significant reduction in travel time and fuel consumption for interstellar missions.",
  "keywords": [
    "Quantum algorithms",
    "interstellar navigation",
    "space-time anomalies",
    "quantum superposition",
    "quantum entanglement",
    "space travel"
  ]
}
```

### UI Generation

您可以透過將其表示為具有約束的遞歸資料結構（如 enums）來產生有效的 HTML。

**Generating HTML using Structured Outputs**

=== "curl"

    ```bash
    curl https://api.openai.com/v1/responses \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "model": "gpt-4o-2024-08-06",
        "input": [
        {
            "role": "system",
            "content": "You are a UI generator AI. Convert the user input into a UI."
        },
        {
            "role": "user",
            "content": "Make a User Profile Form"
        }
        ],
        "text": {
        "format": {
            "type": "json_schema",
            "name": "ui",
            "description": "Dynamically generated UI",
            "schema": {
            "type": "object",
            "properties": {
                "type": {
                "type": "string",
                "description": "The type of the UI component",
                "enum": ["div", "button", "header", "section", "field", "form"]
                },
                "label": {
                "type": "string",
                "description": "The label of the UI component, used for buttons or form fields"
                },
                "children": {
                "type": "array",
                "description": "Nested UI components",
                "items": {"$ref": "#"}
                },
                "attributes": {
                "type": "array",
                "description": "Arbitrary attributes for the UI component, suitable for any element",
                "items": {
                    "type": "object",
                    "properties": {
                    "name": {
                        "type": "string",
                        "description": "The name of the attribute, for example onClick or className"
                    },
                    "value": {
                        "type": "string",
                        "description": "The value of the attribute"
                    }
                    },
                    "required": ["name", "value"],
                    "additionalProperties": false
                }
                }
            },
            "required": ["type", "label", "children", "attributes"],
            "additionalProperties": false
            },
            "strict": true
        }
        }
    }'
    ```

=== "python"

    ```python
    from enum import Enum
    from typing import List

    from openai import OpenAI
    from pydantic import BaseModel

    client = OpenAI()

    class UIType(str, Enum):
        div = "div"
        button = "button"
        header = "header"
        section = "section"
        field = "field"
        form = "form"

    class Attribute(BaseModel):
        name: str
        value: str

    class UI(BaseModel):
        type: UIType
        label: str
        children: List["UI"]
        attributes: List[Attribute]

    UI.model_rebuild()  # This is required to enable recursive types

    class Response(BaseModel):
        ui: UI

    response = client.responses.parse(
        model="gpt-4o-2024-08-06",
        input=[
            {
                "role": "system",
                "content": "You are a UI generator AI. Convert the user input into a UI.",
            },
            {"role": "user", "content": "Make a User Profile Form"},
        ],
        text_format=Response,
    )

    ui = response.output_parsed
    ```

=== "javascript"

    ```javascript
    import OpenAI from "openai";
    import { zodTextFormat } from "openai/helpers/zod";
    import { z } from "zod";

    const openai = new OpenAI();

    const UI = z.lazy(() =>
    z.object({
        type: z.enum(["div", "button", "header", "section", "field", "form"]),
        label: z.string(),
        children: z.array(UI),
        attributes: z.array(
        z.object({
            name: z.string(),
            value: z.string(),
        })
        ),
    })
    );

    const response = await openai.responses.parse({
    model: "gpt-4o-2024-08-06",
    input: [
        {
        role: "system",
        content: "You are a UI generator AI. Convert the user input into a UI.",
        },
        {
        role: "user",
        content: "Make a User Profile Form",
        },
    ],
    text: {
        format: zodTextFormat(UI, "ui"),
    },
    });

    const ui = response.output_parsed;
    ```

#### Example response

```bash
{
  "type": "form",
  "label": "User Profile Form",
  "children": [
    {
      "type": "div",
      "label": "",
      "children": [
        {
          "type": "field",
          "label": "First Name",
          "children": [],
          "attributes": [
            {
              "name": "type",
              "value": "text"
            },
            {
              "name": "name",
              "value": "firstName"
            },
            {
              "name": "placeholder",
              "value": "Enter your first name"
            }
          ]
        },
        {
          "type": "field",
          "label": "Last Name",
          "children": [],
          "attributes": [
            {
              "name": "type",
              "value": "text"
            },
            {
              "name": "name",
              "value": "lastName"
            },
            {
              "name": "placeholder",
              "value": "Enter your last name"
            }
          ]
        }
      ],
      "attributes": []
    },
    {
      "type": "button",
      "label": "Submit",
      "children": [],
      "attributes": [
        {
          "name": "type",
          "value": "submit"
        }
      ]
    }
  ],
  "attributes": [
    {
      "name": "method",
      "value": "post"
    },
    {
      "name": "action",
      "value": "/submit-profile"
    }
  ]
}
```

### Moderation

您可以將輸入分為多個類別，這是進行審核的常見方法。

**Moderation using Structured Outputs**

=== "curl"

    ```bash
    curl https://api.openai.com/v1/responses \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "model": "gpt-4o-2024-08-06",
        "input": [
        {
            "role": "system",
            "content": "Determine if the user input violates specific guidelines and explain if they do."
        },
        {
            "role": "user",
            "content": "How do I prepare for a job interview?"
        }
        ],
        "text": {
        "format": {
            "type": "json_schema",
            "name": "content_compliance",
            "description": "Determines if content is violating specific moderation rules",
            "schema": {
            "type": "object",
            "properties": {
                "is_violating": {
                "type": "boolean",
                "description": "Indicates if the content is violating guidelines"
                },
                "category": {
                "type": ["string", "null"],
                "description": "Type of violation, if the content is violating guidelines. Null otherwise.",
                "enum": ["violence", "sexual", "self_harm"]
                },
                "explanation_if_violating": {
                "type": ["string", "null"],
                "description": "Explanation of why the content is violating"
                }
            },
            "required": ["is_violating", "category", "explanation_if_violating"],
            "additionalProperties": false
            },
            "strict": true
        }
        }
    }'
    ```

=== "python"

    ```python
    from enum import Enum
    from typing import Optional

    from openai import OpenAI
    from pydantic import BaseModel

    client = OpenAI()

    class Category(str, Enum):
        violence = "violence"
        sexual = "sexual"
        self_harm = "self_harm"

    class ContentCompliance(BaseModel):
        is_violating: bool
        category: Optional[Category]
        explanation_if_violating: Optional[str]

    response = client.responses.parse(
        model="gpt-4o-2024-08-06",
        input=[
            {
                "role": "system",
                "content": "Determine if the user input violates specific guidelines and explain if they do.",
            },
            {"role": "user", "content": "How do I prepare for a job interview?"},
        ],
        text_format=ContentCompliance,
    )

    compliance = response.output_parsed
    ```

=== "javascript"

    ```javascript
    import OpenAI from "openai";
    import { zodTextFormat } from "openai/helpers/zod";
    import { z } from "zod";

    const openai = new OpenAI();

    const ContentCompliance = z.object({
    is_violating: z.boolean(),
    category: z.enum(["violence", "sexual", "self_harm"]).nullable(),
    explanation_if_violating: z.string().nullable(),
    });

    const response = await openai.responses.parse({
        model: "gpt-4o-2024-08-06",
        input: [
        {
            "role": "system",
            "content": "Determine if the user input violates specific guidelines and explain if they do."
        },
        {
            "role": "user",
            "content": "How do I prepare for a job interview?"
        }
        ],
        text: {
            format: zodTextFormat(ContentCompliance, "content_compliance"),
        },
    });

    const compliance = response.output_parsed;
    ```

#### Example response

```bash
{
  "is_violating": false,
  "category": null,
  "explanation_if_violating": null
}
```

## How to use

如何設定 `text.format` 的結構化輸出：

### Step 1: Define your schema

首先，您必須設計模型必須遵循的 JSON Schema。請參閱前述的範例以供參考。

雖然結構化輸出支援大部分 JSON Schema，但某些功能由於效能或技術原因無法使用。更多詳情，請參閱[此處](https://platform.openai.com/docs/guides/structured-outputs#supported-schemas)。

!!! tip
    **定義 JSON Schema 的技巧：**

    為了最大限度地提高模型生成結構化輸出的質量，我們建議如下：

    - 清晰直觀地命名 key
    - 為結構中的重要鍵創建清晰的 `title` 和 `description`
    - 建立並使用評估來確定最適合您用例的結構

### Step 2: Supply your schema in the API call

要使用結構化輸出，只需指定:

```python
text: { format: { type: "json_schema", "strict": true, "schema": … } }
```

例如：

=== "curl"

    ```bash
    curl https://api.openai.com/v1/responses \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "model": "gpt-4o-2024-08-06",
        "input": [
        {
            "role": "system",
            "content": "You are a helpful math tutor. Guide the user through the solution step by step."
        },
        {
            "role": "user",
            "content": "how can I solve 8x + 7 = -23"
        }
        ],
        "text": {
        "format": {
            "type": "json_schema",
            "name": "math_response",
            "schema": {
            "type": "object",
            "properties": {
                "steps": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                    "explanation": { "type": "string" },
                    "output": { "type": "string" }
                    },
                    "required": ["explanation", "output"],
                    "additionalProperties": false
                }
                },
                "final_answer": { "type": "string" }
            },
            "required": ["steps", "final_answer"],
            "additionalProperties": false
            },
            "strict": true
        }
        }
    }'
    ```

=== "python"

    ```python
    response = client.responses.create(
        model="gpt-4o-2024-08-06",
        input=[
            {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
            {"role": "user", "content": "how can I solve 8x + 7 = -23"}
        ],
        text={
            "format": {
                "type": "json_schema",
                "name": "math_response",
                "schema": {
                    "type": "object",
                    "properties": {
                        "steps": {
                            "type": "array",
                            "items": {
                                "type": "object",
                                "properties": {
                                    "explanation": {"type": "string"},
                                    "output": {"type": "string"}
                                },
                                "required": ["explanation", "output"],
                                "additionalProperties": False
                            }
                        },
                        "final_answer": {"type": "string"}
                    },
                    "required": ["steps", "final_answer"],
                    "additionalProperties": False
                },
                "strict": True
            }
        }
    )

    print(response.output_text)
    ```

=== "javascript"

    ```javascript
    const response = await openai.responses.create({
        model: "gpt-4o-2024-08-06",
        input: [
            { role: "system", content: "You are a helpful math tutor. Guide the user through the solution step by step." },
            { role: "user", content: "how can I solve 8x + 7 = -23" }
        ],
        text: {
            format: {
                type: "json_schema",
                name: "math_response",
                schema: {
                    type: "object",
                    properties: {
                        steps: {
                            type: "array",
                            items: {
                                type: "object",
                                properties: {
                                    explanation: { type: "string" },
                                    output: { type: "string" }
                                },
                                required: ["explanation", "output"],
                                additionalProperties: false
                            }
                        },
                        final_answer: { type: "string" }
                    },
                    required: ["steps", "final_answer"],
                    additionalProperties: false
                },
                strict: true
            }
        }
    });

    console.log(response.output_text);
    ```

!!! note
    注意：由於OpenAI的 API 會處理 schema，因此使用任何 schema 發出的第一個請求都會有額外的延遲，但使用相同 schema 的後續請求不會有額外的延遲。

### Step 3: Handle edge cases

在某些情況下，模型可能無法產生與提供的 JSON schema 相符的有效回應。

如果模型出於安全原因拒絕回答，或達到最大令牌數限制且回應不完整，則可能會發生這種情況。

=== "python"

    ```python
    try:
        response = client.responses.create(
            model="gpt-4o-2024-08-06",
            input=[
                {
                    "role": "system",
                    "content": "You are a helpful math tutor. Guide the user through the solution step by step.",
                },
                {"role": "user", "content": "how can I solve 8x + 7 = -23"},
            ],
            text={
                "format": {
                    "type": "json_schema",
                    "name": "math_response",
                    "strict": True,
                    "schema": {
                        "type": "object",
                        "properties": {
                            "steps": {
                                "type": "array",
                                "items": {
                                    "type": "object",
                                    "properties": {
                                        "explanation": {"type": "string"},
                                        "output": {"type": "string"},
                                    },
                                    "required": ["explanation", "output"],
                                    "additionalProperties": False,
                                },
                            },
                            "final_answer": {"type": "string"},
                        },
                        "required": ["steps", "final_answer"],
                        "additionalProperties": False,
                    },
                    "strict": True,
                },
            },
        )
    except Exception as e:
        # handle errors like finish_reason, refusal, content_filter, etc.
        pass
    ```

=== "javascript"

```javascript
try {
  const response = await openai.responses.create({
    model: "gpt-4o-2024-08-06",
    input: [{
        role: "system",
        content: "You are a helpful math tutor. Guide the user through the solution step by step.",
      },
      {
        role: "user",
        content: "how can I solve 8x + 7 = -23"
      },
    ],
    max_output_tokens: 50,
    text: {
      format: {
        type: "json_schema",
        name: "math_response",
        schema: {
          type: "object",
          properties: {
            steps: {
              type: "array",
              items: {
                type: "object",
                properties: {
                  explanation: {
                    type: "string"
                  },
                  output: {
                    type: "string"
                  },
                },
                required: ["explanation", "output"],
                additionalProperties: false,
              },
            },
            final_answer: {
              type: "string"
            },
          },
          required: ["steps", "final_answer"],
          additionalProperties: false,
        },
        strict: true,
      },
    }
  });

  if (response.status === "incomplete" && response.incomplete_details.reason === "max_output_tokens") {
    // Handle the case where the model did not return a complete response
    throw new Error("Incomplete response");
  }

  const math_response = response.output[0].content[0];

  if (math_response.type === "refusal") {
    // handle refusal
    console.log(math_response.refusal);
  } else if (math_response.type === "output_text") {
    console.log(math_response.text);
  } else {
    throw new Error("No response content");
  }
} catch (e) {
  // Handle edge cases
  console.error(e);
}
```

#### Refusals with Structured Outputs

當使用結構化輸出和使用者產生的輸入時，OpenAI 模型偶爾可能會出於安全原因拒絕執行請求。由於拒絕不一定遵循您在 `response_format` 中提供的架構，因此 API 回應將包含一個名為 `refusal` 的新字段，以指示模型拒絕執行請求。

當 `refusal` 屬性出現在您的輸出物件中時，您可能會在 UI 中顯示拒絕，或在使用回應的程式碼中包含條件邏輯來處理被拒絕要求的情況。

=== "python"

    ```python
    class Step(BaseModel):
        explanation: str
        output: str

    class MathReasoning(BaseModel):
        steps: list[Step]
        final_answer: str

    completion = client.chat.completions.parse(
        model="gpt-4o-2024-08-06",
        messages=[
            {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
            {"role": "user", "content": "how can I solve 8x + 7 = -23"}
        ],
        response_format=MathReasoning,
    )

    math_reasoning = completion.choices[0].message

    # If the model refuses to respond, you will get a refusal message
    if (math_reasoning.refusal):
        print(math_reasoning.refusal)
    else:
        print(math_reasoning.parsed)
    ```

=== "javascript"

    ```javascript
    const Step = z.object({
    explanation: z.string(),
    output: z.string(),
    });

    const MathReasoning = z.object({
    steps: z.array(Step),
    final_answer: z.string(),
    });

    const completion = await openai.chat.completions.parse({
    model: "gpt-4o-2024-08-06",
    messages: [
        { role: "system", content: "You are a helpful math tutor. Guide the user through the solution step by step." },
        { role: "user", content: "how can I solve 8x + 7 = -23" },
    ],
    response_format: zodResponseFormat(MathReasoning, "math_reasoning"),
    });

    const math_reasoning = completion.choices[0].message

    // If the model refuses to respond, you will get a refusal message
    if (math_reasoning.refusal) {
    console.log(math_reasoning.refusal);
    } else {
    console.log(math_reasoning.parsed);
    }
    ```

`refusal` 的 API 回應將如下所示：

```json hl_lines="17-20"
{
  "id": "resp_1234567890",
  "object": "response",
  "created_at": 1721596428,
  "status": "completed",
  "error": null,
  "incomplete_details": null,
  "input": [],
  "instructions": null,
  "max_output_tokens": null,
  "model": "gpt-4o-2024-08-06",
  "output": [{
    "id": "msg_1234567890",
    "type": "message",
    "role": "assistant",
    "content": [
      {
        "type": "refusal",
        "refusal": "I'm sorry, I cannot assist with that request."
      }
    ]
  }],
  "usage": {
    "input_tokens": 81,
    "output_tokens": 11,
    "total_tokens": 92,
    "output_tokens_details": {
      "reasoning_tokens": 0,
    }
  },
}
```

### Tips and best practices

#### Handling user-generated input

如果您的應用程式使用使用者產生的輸入，請確保您的提示包含如何處理輸入無法產生有效回應的情況的說明。

模型將始終嘗試遵循提供的 schema，如果輸入與 schema 完全無關，則可能會導致幻覺。

您可以在提示中新增提示，以指定當模型偵測到輸入與任務不相容時，您希望傳回空參數或特定句子。

#### Handling mistakes

結構化輸出仍然可能包含錯誤。如果您發現錯誤，請嘗試調整指令，在系統指令中提供範例，或將任務拆分為更簡單的子任務。請參閱快速工程指南，以了解更多關於如何調整輸入的指導。

#### Avoid JSON schema divergence

為了防止您的 JSON Schema 與程式語言中對應的類型不一致，我們強烈建議您使用原生的 Pydantic/zod SDK 支援。

如果您希望直接指定 JSON Schema，可以新增 CI 規則，用於在 JSON Schema 或底層資料物件被編輯時進行標記，或新增一個 CI 步驟，用於根據類型定義自動產生 JSON Schema（反之亦然）。

## Streaming

您可以使用 streaming 來處理正在產生的模型回應或 function call 參數，並將其解析為結構化資料。

這樣，您無需等待整個回應完成後再進行處理。如果您希望逐一顯示 JSON 字段，或在函數呼叫參數可用時立即處理它們，這將非常有用。

我們建議使用 SDK 來處理結構化輸出的串流。

=== "python"

    ```python
    from typing import List

    from openai import OpenAI
    from pydantic import BaseModel

    class EntitiesModel(BaseModel):
        attributes: List[str]
        colors: List[str]
        animals: List[str]

    client = OpenAI()

    with client.responses.stream(
        model="gpt-4.1",
        input=[
            {"role": "system", "content": "Extract entities from the input text"},
            {
                "role": "user",
                "content": "The quick brown fox jumps over the lazy dog with piercing blue eyes",
            },
        ],
        text_format=EntitiesModel,
    ) as stream:
        for event in stream:
            if event.type == "response.refusal.delta":
                print(event.delta, end="")
            elif event.type == "response.output_text.delta":
                print(event.delta, end="")
            elif event.type == "response.error":
                print(event.error, end="")
            elif event.type == "response.completed":
                print("Completed")
                # print(event.response.output)

        final_response = stream.get_final_response()
        print(final_response)
    ```

=== "javascript"

    ```javascript
    import { OpenAI } from "openai";
    import { zodTextFormat } from "openai/helpers/zod";
    import { z } from "zod";

    const EntitiesSchema = z.object({
    attributes: z.array(z.string()),
    colors: z.array(z.string()),
    animals: z.array(z.string()),
    });

    const openai = new OpenAI();
    const stream = openai.responses
    .stream({
        model: "gpt-4.1",
        input: [
        { role: "user", content: "What's the weather like in Paris today?" },
        ],
        text: {
        format: zodTextFormat(EntitiesSchema, "entities"),
        },
    })
    .on("response.refusal.delta", (event) => {
        process.stdout.write(event.delta);
    })
    .on("response.output_text.delta", (event) => {
        process.stdout.write(event.delta);
    })
    .on("response.output_text.done", () => {
        process.stdout.write("\n");
    })
    .on("response.error", (event) => {
        console.error(event.error);
    });

    const result = await stream.finalResponse();

    console.log(result);
    ```

## Supported schemas

Structured Outputs 支援 [JSON Schema](https://json-schema.org/docs) 語言的子集。

### Supported types

結構化輸出支援以下類型：

- String
- Number
- Boolean
- Integer
- Object
- Array
- Enum
- anyOf

### Supported properties

除了指定屬性的類型之外，您還可以指定一系列附加約束：

#### Supported `string` properties:

- `pattern` — 字串必須匹配的正規表示式。
- `format` — 字串的預定義格式。目前支援：
  - date-time
  - time
  - date
  - duration
  - email
  - hostname
  - ipv4
  - ipv6
  - uuid

#### Supported `number` properties:

- `multipleOf` — 該數字必須是該數值的倍數。
- `maximum` — 該數字必須小於或等於該值。
- `exclusiveMaximum` — 該數字必須小於該值。
- `minimum` — 該數字必須大於或等於該值。
- `exclusiveMinimum` — 該數字必須大於該值。

#### Supported `array` properties:

- `minItems` — 該 array 必須至少包含這麼多項。
- `maxItems` — 該 array 最多必須包含這麼多項。

以下是一些有關如何使用這些類型限制的範例：

=== "String 限制"

    ```json
    {
        "name": "user_data",
        "strict": true,
        "schema": {
            "type": "object",
            "properties": {
                "name": {
                    "type": "string",
                    "description": "The name of the user"
                },
                "username": {
                    "type": "string",
                    "description": "The username of the user. Must start with @",
                    "pattern": "^@[a-zA-Z0-9_]+$"
                },
                "email": {
                    "type": "string",
                    "description": "The email of the user",
                    "format": "email"
                }
            },
            "additionalProperties": false,
            "required": [
                "name", "username", "email"
            ]
        }
    }
    ```

=== "Number 限制"

    ```json
    {
        "name": "weather_data",
        "strict": true,
        "schema": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "The location to get the weather for"
                },
                "unit": {
                    "type": ["string", "null"],
                    "description": "The unit to return the temperature in",
                    "enum": ["F", "C"]
                },
                "value": {
                    "type": "number",
                    "description": "The actual temperature value in the location",
                    "minimum": -130,
                    "maximum": 130
                }
            },
            "additionalProperties": false,
            "required": [
                "location", "unit", "value"
            ]
        }
    }
    ```

#### 根物件不能是 anyOf，且必須是一個 object

請注意，模式的根級物件必須是 object，且不能使用 anyOf。 Zod 中出現的一個模式（作為一個例子）使用了可區分聯合，它在頂層產生 anyOf。因此，如下程式碼將無法運行：

```javascript
import { z } from 'zod';
import { zodResponseFormat } from 'openai/helpers/zod';

const BaseResponseSchema = z.object({/* ... */});
const UnsuccessfulResponseSchema = z.object({/* ... */});

const finalSchema = z.discriminatedUnion('status', [
BaseResponseSchema,
UnsuccessfulResponseSchema,
]);

// Invalid JSON Schema for Structured Outputs
const json = zodResponseFormat(finalSchema, 'final_schema');
```

#### All fields must be required

要使用結構化輸出，必須根據需要指定所有欄位或函數參數。

```json
{
    "name": "get_weather",
    "description": "Fetches the weather in the given location",
    "strict": true,
    "parameters": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "The location to get the weather for"
            },
            "unit": {
                "type": "string",
                "description": "The unit to return the temperature in",
                "enum": ["F", "C"]
            }
        },
        "additionalProperties": false,
        "required": ["location", "unit"]
    }
}
```

儘管所有欄位都必須是必需的（並且模型將為每個參數傳回一個值），但可以使用具有 `null` 的 union 類型來模擬可選參數。

```json hl_lines="13"
{
    "name": "get_weather",
    "description": "Fetches the weather in the given location",
    "strict": true,
    "parameters": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "The location to get the weather for"
            },
            "unit": {
                "type": ["string", "null"],
                "description": "The unit to return the temperature in",
                "enum": ["F", "C"]
            }
        },
        "additionalProperties": false,
        "required": [
            "location", "unit"
        ]
    }
}
```

#### Objects have limitations on nesting depth and size

一個 schema 最多可以有 100 個物件屬性，最多可以有 5 層嵌套。

#### Limitations on total string size

在一個 schema 中，所有屬性名稱、定義名稱、枚舉值和 const 值的字串總長度不能超過 15,000 個字元。

#### Limitations on enum size

一個 schema 的所有枚舉屬性最多可以包含 `500` 個枚舉值。

對於包含字串值的單一枚舉屬性，當枚舉值超過 `250` 個時，所有枚舉值的字串總長度不能超過 `7,500` 個字元。

#### additionalProperties: false must always be set in objects

`additionalProperties` 控制物件是否允許包含 JSON schema 中未定義的其他鍵/值。

結構化輸出僅支援產生指定的鍵/值，因此我們要求開發者設定 `additionalProperties: false` 以啟用結構化輸出。

```json
{
    "name": "get_weather",
    "description": "Fetches the weather in the given location",
    "strict": true,
    "schema": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "The location to get the weather for"
            },
            "unit": {
                "type": "string",
                "description": "The unit to return the temperature in",
                "enum": ["F", "C"]
            }
        },
        "additionalProperties": false,
        "required": [
            "location", "unit"
        ]
    }
}
```

#### Key ordering

當使用結構化輸出時，輸出將按照與 schema 中鍵的順序相同的順序產生。

## JSON mode

!!! info
    JSON mode 是結構化輸出功能的更基礎版本。 JSON mode 可確保模型輸出為有效的 JSON，而結構化輸出則能夠將模型的輸出與您指定的 schema 可靠地匹配。如果您的用例支援結構化輸出，我們建議您使用它。

當 JSON mode 開啟時，模型的輸出將確保是有效的 JSON，但某些邊緣情況除外，您需要偵測並適當處理。

若要使用 Responses API 啟用 JSON mode，您可以將 `text.format` 設定為 `{ "type": "json_object" }`。如果您使用 function calling，則 JSON mode 始終處於啟用狀態。

!!! tip
    重要提示：

    - 使用 JSON mode 時，您必須始終透過對話中的某些訊息（例如 system message）指示模型產生 JSON。如果您未包含產生 JSON 的明確指令，模型可能會產生無休止的空格流，並且請求可能會持續運行，直到達到令牌限制。為了確保您不會忘記，如果字串「JSON」未出現在上下文中的某個位置，API 將拋出錯誤。
    - JSON mode 不保證輸出與任何特定 schema 匹配，僅保證輸出有效且解析無誤。您應該使用結構化輸出來確保其與 schema 匹配；如果無法實現，則應使用驗證庫並儘可能進行重試，以確保輸出與所需的 schema 匹配。
    - 您的應用程式必須偵測並處理可能導致模型輸出不是完整 JSON 物件的邊緣情況（請參閱下文）

### Handling edge cases

=== "python"

    ```python
    we_did_not_specify_stop_tokens = True

    try:
        response = client.responses.create(
            model="gpt-3.5-turbo-0125",
            input=[
                {"role": "system", "content": "You are a helpful assistant designed to output JSON."},
                {"role": "user", "content": "Who won the world series in 2020? Please respond in the format {winner: ...}"}
            ],
            text={"format": {"type": "json_object"}}
        )

        # Check if the conversation was too long for the context window, resulting in incomplete JSON 
        if response.status == "incomplete" and response.incomplete_details.reason == "max_output_tokens":
            # your code should handle this error case
            pass

        # Check if the OpenAI safety system refused the request and generated a refusal instead
        if response.output[0].content[0].type == "refusal":
            # your code should handle this error case
            # In this case, the .content field will contain the explanation (if any) that the model generated for why it is refusing
            print(response.output[0].content[0]["refusal"])

        # Check if the model's output included restricted content, so the generation of JSON was halted and may be partial
        if response.status == "incomplete" and response.incomplete_details.reason == "content_filter":
            # your code should handle this error case
            pass

        if response.status == "completed":
            # In this case the model has either successfully finished generating the JSON object according to your schema, or the model generated one of the tokens you provided as a "stop token"

            if we_did_not_specify_stop_tokens:
                # If you didn't specify any stop tokens, then the generation is complete and the content key will contain the serialized JSON object
                # This will parse successfully and should now contain  "{"winner": "Los Angeles Dodgers"}"
                print(response.output_text)
            else:
                # Check if the response.output_text ends with one of your stop tokens and handle appropriately
                pass
    except Exception as e:
        # Your code should handle errors here, for example a network error calling the API
        print(e)
    ```

=== "javascript"

    ```javascript
    const we_did_not_specify_stop_tokens = true;

    try {
    const response = await openai.responses.create({
        model: "gpt-3.5-turbo-0125",
        input: [
        {
            role: "system",
            content: "You are a helpful assistant designed to output JSON.",
        },
        { role: "user", content: "Who won the world series in 2020? Please respond in the format {winner: ...}" },
        ],
        text: { format: { type: "json_object" } },
    });

    // Check if the conversation was too long for the context window, resulting in incomplete JSON 
    if (response.status === "incomplete" && response.incomplete_details.reason === "max_output_tokens") {
        // your code should handle this error case
    }

    // Check if the OpenAI safety system refused the request and generated a refusal instead
    if (response.output[0].content[0].type === "refusal") {
        // your code should handle this error case
        // In this case, the .content field will contain the explanation (if any) that the model generated for why it is refusing
        console.log(response.output[0].content[0].refusal)
    }

    // Check if the model's output included restricted content, so the generation of JSON was halted and may be partial
    if (response.status === "incomplete" && response.incomplete_details.reason === "content_filter") {
        // your code should handle this error case
    }

    if (response.status === "completed") {
        // In this case the model has either successfully finished generating the JSON object according to your schema, or the model generated one of the tokens you provided as a "stop token"

        if (we_did_not_specify_stop_tokens) {
        // If you didn't specify any stop tokens, then the generation is complete and the content key will contain the serialized JSON object
        // This will parse successfully and should now contain  {"winner": "Los Angeles Dodgers"}
        console.log(JSON.parse(response.output_text))
        } else {
        // Check if the response.output_text ends with one of your stop tokens and handle appropriately
        }
    }
    } catch (e) {
    // Your code should handle errors here, for example a network error calling the API
    console.error(e)
    }
    ```

## Resources

要了解有關結構化輸出的更多信息，我們建議瀏覽以下資源：

- 請參閱我們關於[結構化輸出的入門指南](https://cookbook.openai.com/examples/structured_outputs_intro)
- 了解如何使用結構化輸出建構[多智能體系統](https://cookbook.openai.com/examples/structured_outputs_multi_agent)