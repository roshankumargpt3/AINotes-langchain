# Prompting Foundations with LangChain

## Source

This summary is based on:

```text
https://github.com/prudhviakella/vs-langchain-agents/blob/main/01-agents-foundations/1_1_prompting.ipynb
```

---

## Key Concepts

### 1. Basic Prompting

A basic prompt sends a user message to the model without additional instructions.

It is useful for:

- Testing default model behavior
- Asking simple questions
- Creating a baseline for comparison

---

### 2. System Prompts

A system prompt defines the model's:

- Role
- Personality
- Task
- Behavior
- Response constraints

Example instruction:

```text
You are a science fiction writer.
Create fictional capital cities when requested.
```

System prompts provide more control than basic prompts.

---

### 3. Few-Shot Prompting

Few-shot prompting provides examples that demonstrate the expected response style or format.

For example:

```text
User: What is the capital of Mars?
Writer: Marsialis

User: What is the capital of Venus?
Writer: Venusovia
```

The model can use these examples to infer that it should return short, fictional city names.

Few-shot prompting is useful for:

- Consistent formatting
- Specific writing styles
- Naming conventions
- Classification patterns
- Short, predictable answers

---

### 4. Structured Prompts

A structured prompt asks the model to follow a text-based template.

Example:

```text
Name: The name of the city
Location: Where it is located
Vibe: Two or three words describing the atmosphere
Economy: Main industries
```

Structured prompts improve readability, but the response is still plain text and may require manual parsing.

---

### 5. Structured Output

Structured output uses a schema to define the expected response fields and data types.

This notebook uses Pydantic to define the schema.

Benefits include:

- Typed fields
- Easier access from Python
- Less manual string parsing
- Better integration with APIs and databases
- Validation of returned data

The structured result is accessed through:

```python
response["structured_response"]
```

---

### 6. Pydantic Models

Pydantic defines the expected structure of the model response.

```python
class CapitalInfo(BaseModel):
    name: str
    location: str
    vibe: str
    economy: str
```

Each field has a name and a Python type.

---

## Combined Example

The following example demonstrates all concepts in one script:

1. Environment variable loading
2. Basic prompting
3. System prompts
4. Few-shot prompting
5. Structured text prompts
6. Pydantic structured output
7. Accessing typed response fields

```python
from dotenv import load_dotenv
from langchain.agents import create_agent
from langchain.messages import HumanMessage
from pydantic import BaseModel


# ------------------------------------------------------------
# 1. Load environment variables
# ------------------------------------------------------------

# Loads values such as OPENAI_API_KEY from a .env file.
load_dotenv()


# ------------------------------------------------------------
# Shared user question
# ------------------------------------------------------------

question = HumanMessage(
    content="What is the capital of the moon?"
)


# ------------------------------------------------------------
# 2. Basic prompting
# ------------------------------------------------------------

basic_agent = create_agent(
    model="gpt-5-nano"
)

basic_response = basic_agent.invoke(
    {"messages": [question]}
)

print("\n=== BASIC PROMPT ===")
print(basic_response["messages"][1].content)


# ------------------------------------------------------------
# 3. System prompt
# ------------------------------------------------------------

system_agent = create_agent(
    model="gpt-5-nano",
    system_prompt="""
    You are a science fiction writer.
    Create a fictional capital city whenever the user requests one.
    """
)

system_response = system_agent.invoke(
    {"messages": [question]}
)

print("\n=== SYSTEM PROMPT ===")
print(system_response["messages"][1].content)


# ------------------------------------------------------------
# 4. Few-shot prompting
# ------------------------------------------------------------

few_shot_agent = create_agent(
    model="gpt-5-nano",
    system_prompt="""
    You are a science fiction writer.
    Create a fictional space capital city when requested.

    Follow the examples below:

    User: What is the capital of Mars?
    Scifi Writer: Marsialis

    User: What is the capital of Venus?
    Scifi Writer: Venusovia

    Return only the fictional city name.
    """
)

few_shot_response = few_shot_agent.invoke(
    {"messages": [question]}
)

print("\n=== FEW-SHOT PROMPT ===")
print(few_shot_response["messages"][1].content)


# ------------------------------------------------------------
# 5. Structured text prompt
# ------------------------------------------------------------

structured_prompt_agent = create_agent(
    model="gpt-5-nano",
    system_prompt="""
    You are a science fiction writer.
    Create a fictional capital city when requested.

    Use exactly the following text structure:

    Name: The name of the capital city
    Location: Where it is based
    Vibe: Two or three words describing its atmosphere
    Economy: Main industries
    """
)

structured_prompt_response = structured_prompt_agent.invoke(
    {"messages": [question]}
)

print("\n=== STRUCTURED TEXT PROMPT ===")
print(structured_prompt_response["messages"][1].content)


# ------------------------------------------------------------
# 6. Structured output with Pydantic
# ------------------------------------------------------------

class CapitalInfo(BaseModel):
    name: str
    location: str
    vibe: str
    economy: str


structured_output_agent = create_agent(
    model="gpt-5-nano",
    system_prompt="""
    You are a science fiction writer.
    Create a fictional capital city for the location requested by the user.
    Make the response creative and concise.
    """,
    response_format=CapitalInfo
)

structured_output_response = structured_output_agent.invoke(
    {"messages": [question]}
)


# ------------------------------------------------------------
# 7. Access structured fields
# ------------------------------------------------------------

capital_info = structured_output_response["structured_response"]

print("\n=== STRUCTURED OUTPUT ===")
print(f"Name: {capital_info.name}")
print(f"Location: {capital_info.location}")
print(f"Vibe: {capital_info.vibe}")
print(f"Economy: {capital_info.economy}")


# Structured output can also be converted to a dictionary.
capital_data = capital_info.model_dump()

print("\n=== AS DICTIONARY ===")
print(capital_data)
```

---

## Prompting Comparison

| Technique | Output type | Main purpose |
|---|---|---|
| Basic prompt | Free-form text | Simple questions and experimentation |
| System prompt | Controlled free-form text | Define role, behavior, and task |
| Few-shot prompt | Pattern-based text | Demonstrate style or format |
| Structured prompt | Template-like text | Produce human-readable fields |
| Structured output | Typed Python object | Use model output in application code |

---

## Key Packages Used

### `langchain`

Used to create and invoke agents.

```python
from langchain.agents import create_agent
```

Important features used:

```python
create_agent(...)
agent.invoke(...)
system_prompt=...
response_format=...
```

---

### `langchain.messages`

Used to create user messages.

```python
from langchain.messages import HumanMessage
```

Example:

```python
question = HumanMessage(
    content="What is the capital of the moon?"
)
```

---

### `pydantic`

Used to define and validate structured output.

```python
from pydantic import BaseModel
```

Example:

```python
class CapitalInfo(BaseModel):
    name: str
    location: str
    vibe: str
    economy: str
```

---

### `python-dotenv`

Used to load environment variables from a `.env` file.

```python
from dotenv import load_dotenv

load_dotenv()
```

Example `.env` file:

```text
OPENAI_API_KEY=your-api-key
```

API keys should not be hardcoded or committed to Git.

---

## Installation

The packages directly imported by the notebook can be installed with:

```bash
pip install -U langchain pydantic python-dotenv
```

Depending on the configured model provider and LangChain version, an additional provider integration package may be required.

---

## Main Takeaways

1. Start with a basic prompt to observe default behavior.
2. Use a system prompt to define the model's role and rules.
3. Use few-shot examples to demonstrate the desired style.
4. Use structured prompts when humans need readable formatted text.
5. Use Pydantic structured output when software needs reliable fields.
6. Prefer typed structured output over manually parsing model-generated text.
7. Keep examples consistent and free from contradictory instructions.
8. Test prompts with multiple inputs before using them in production.
