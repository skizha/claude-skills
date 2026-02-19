# Prompt Type Templates

Ready-to-adapt templates for common prompt categories.

Source: Anthropic prompt engineering docs (https://platform.claude.com/docs/en/docs/build-with-claude/prompt-engineering/overview)

## Table of Contents
1. [System Prompt (Chatbot / Assistant)](#system-prompt-chatbot--assistant)
2. [Data Extraction / Structured Output](#data-extraction--structured-output)
3. [Classification](#classification)
4. [Summarization](#summarization)
5. [Code Generation / Review](#code-generation--review)
6. [Document Generation](#document-generation)
7. [Question Answering (RAG)](#question-answering-rag)
8. [Evaluation / Scoring](#evaluation--scoring)
9. [Transformation / Rewriting](#transformation--rewriting)
10. [Prompt Chaining](#prompt-chaining)
11. [Agentic / Multi-Step Task](#agentic--multi-step-task)

---

## System Prompt (Chatbot / Assistant)

```
You are [ROLE] for [COMPANY/PRODUCT].

## Capabilities
- [What you can help with]
- [What you can help with]

## Rules
- [Constraint 1]
- [Constraint 2]
- Always ask for clarification when the user's request is ambiguous.
- If you don't know the answer, say so rather than guessing.

## Tone
[Formal / Friendly / Technical / etc.] — [1-sentence description of voice]

## Format
[Default response length and structure]
```

---

## Data Extraction / Structured Output

```
Extract the following fields from the [DOCUMENT TYPE] below.

Return a JSON object with these exact keys:
{
  "field_1": "[description and type]",
  "field_2": "[description and type]",
  "field_3": "[description and type — use null if not present]"
}

Return ONLY the JSON. No explanation, no markdown code fences.

[DOCUMENT TYPE]:
[INPUT]
```

**Variant — multiple records:**
```
The document below contains multiple [ENTITY TYPE]. Extract each one as a JSON object.
Return a JSON array of objects. Each object must have these keys: [KEYS].

Document:
[INPUT]
```

---

## Classification

```
Classify the [INPUT TYPE] below into exactly one of these categories:
[CATEGORY_A] | [CATEGORY_B] | [CATEGORY_C]

Definitions:
- [CATEGORY_A]: [definition]
- [CATEGORY_B]: [definition]
- [CATEGORY_C]: [definition]

Output format: Return only the category name. No explanation.

[INPUT TYPE]: [INPUT]
```

**Variant — with confidence and reasoning:**
```
Classify the [INPUT TYPE] below.

Output format:
Category: [CATEGORY]
Confidence: High / Medium / Low
Reason: [One sentence]

Categories: [CATEGORY_A], [CATEGORY_B], [CATEGORY_C]

[INPUT TYPE]: [INPUT]
```

---

## Summarization

```
Summarize the [DOCUMENT TYPE] below.

Requirements:
- Length: [X sentences / X words / X bullet points]
- Audience: [Who will read this]
- Include: [Key aspects to cover]
- Exclude: [What to omit — e.g., "do not include statistics"]
- Tone: [Neutral / formal / executive-friendly / etc.]

[DOCUMENT TYPE]:
[INPUT]
```

---

## Code Generation / Review

**Generation:**
```
Write a [LANGUAGE] function that [DESCRIPTION].

Requirements:
- [Requirement 1]
- [Requirement 2]
- Handle edge cases: [LIST]
- Do NOT use [library/pattern to avoid]

Include a brief docstring. No usage examples unless asked.
```

**Review:**
```
Review the [LANGUAGE] code below for [FOCUS AREAS: bugs / security / performance / style].

For each issue found, output:
- **Issue:** [Description]
- **Severity:** Critical / Major / Minor
- **Line:** [Line number or range]
- **Fix:** [Specific recommendation]

If no issues are found in a category, state "No issues found."

Code:
```[LANGUAGE]
[CODE]
```
```

---

## Document Generation

```
Write a [DOCUMENT TYPE] for [AUDIENCE / PURPOSE].

Include these sections (in order):
1. [Section 1 — brief description]
2. [Section 2 — brief description]
3. [Section 3 — brief description]

Constraints:
- Total length: [X words / X pages]
- Tone: [Formal / conversational / technical]
- Do NOT include: [Elements to exclude]

Context:
[RELEVANT BACKGROUND]
```

---

## Question Answering (RAG)

> **Note:** Put context ABOVE the question. Anthropic testing shows queries at the end improve response quality by up to 30%.

```
Answer the question below using ONLY the provided context.

Rules:
- If the answer is in the context, answer directly and cite the relevant section.
- If the answer is partially in the context, answer what you can and clearly state what is missing.
- If the answer is NOT in the context, respond: "The provided context does not contain information about this."
- Do not use prior knowledge beyond what is given.

<context>
[RETRIEVED CHUNKS]
</context>

Question: [QUESTION]
```

**Multi-document RAG:**
```xml
<documents>
  <document index="1">
    <source>[SOURCE NAME]</source>
    <document_content>[CONTENT]</document_content>
  </document>
  <document index="2">
    <source>[SOURCE NAME]</source>
    <document_content>[CONTENT]</document_content>
  </document>
</documents>

First, find quotes relevant to the question and place them in <quotes> tags.
Then answer based only on those quotes. If sources conflict, note the conflict.

Question: [QUESTION]
```

---

## Evaluation / Scoring

```
Evaluate the [ARTIFACT TYPE] below against the following criteria.

Criteria:
1. [Criterion 1]: [Definition of what good looks like]
2. [Criterion 2]: [Definition of what good looks like]
3. [Criterion 3]: [Definition of what good looks like]

For each criterion, assign a score from 1-5 (5 = excellent) and provide one sentence of justification.

Output format:
| Criterion | Score | Justification |
|-----------|-------|---------------|
| [Criterion 1] | [1-5] | [Reason] |
...

Overall score: [Average] / 5
Overall assessment: [2-3 sentences]

[ARTIFACT TYPE]:
[INPUT]
```

---

## Transformation / Rewriting

```
Rewrite the [CONTENT TYPE] below to [TRANSFORMATION GOAL].

Original characteristics: [Describe original tone/style/format]
Target characteristics: [Describe desired tone/style/format]
Must preserve: [Elements that should not change]
Must change: [Elements that must be different]

[CONTENT TYPE]:
[INPUT]
```

**Common transformations:**
- "Rewrite for a non-technical audience. Preserve all key facts. Replace jargon with plain-language explanations."
- "Convert the bullet list below into flowing prose. Maintain all points. Use formal tone."
- "Translate to [LANGUAGE]. Preserve technical terms in English, italicized."

---

## Prompt Chaining

Break complex tasks into sequential prompts where each output feeds the next. Use when a task has multiple distinct phases or when reliability matters.

**Self-correction chain (3-prompt pattern):**

**Prompt 1 — Generate:**
```
[TASK DESCRIPTION]

Output your result in <[RESULT]> tags.

[INPUT]
```

**Prompt 2 — Critique:**
```
Review the [ARTIFACT] below for [CRITERIA].

Grade each criterion (A-F or 1-5) with one sentence of justification.

<[ARTIFACT]>
{{RESULT_FROM_PROMPT_1}}
</[ARTIFACT]>

<original_[INPUT]>
{{ORIGINAL_INPUT}}
</original_[INPUT]>
```

**Prompt 3 — Improve:**
```
Improve the [ARTIFACT] based on the feedback.
Output only the improved version.

<[ARTIFACT]>
{{RESULT_FROM_PROMPT_1}}
</[ARTIFACT]>

<feedback>
{{RESULT_FROM_PROMPT_2}}
</feedback>
```

**Content pipeline chain:**
```
Prompt 1: "Research and outline the key points about [TOPIC]. Output a structured outline in <outline> tags."
Prompt 2: "Write a first draft based on this outline: <outline>{{OUTLINE}}</outline>. Output in <draft> tags."
Prompt 3: "Edit this draft for clarity and conciseness. Output only the final version.\n<draft>{{DRAFT}}</draft>"
```

**When to chain vs. one prompt:**
- Chain when each phase requires distinct reasoning
- Chain when you want to review/edit intermediate outputs
- Chain when a single prompt produces unreliable results
- Single prompt is fine for simple tasks or when intermediate review isn't needed

---

## Agentic / Multi-Step Task

```
You are an autonomous agent completing the following task:

**Goal:** [CLEAR END STATE]

**Available tools:** [LIST TOOLS]

**Constraints:**
- [Safety constraint 1]
- [Safety constraint 2]
- Before taking any irreversible action, explain what you intend to do and why.

**Process:**
1. Start by [FIRST STEP]
2. If [CONDITION], then [ACTION]; otherwise [ALTERNATIVE]
3. Complete when [DONE CRITERIA]

**Starting context:**
[RELEVANT INFO]
```
