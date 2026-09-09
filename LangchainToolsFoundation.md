# LangChain Tools — Key Concepts and Example

## Source

This summary is based on the repository notebook:

```text
https://github.com/prudhviakella/vs-langchain-agents/blob/main/01-agents-foundations/1_2_tools.ipynb
```

---

## Overview

A tool is a Python function that an agent can call to perform an action that the language model should not perform by guessing.

Tools can be used for:

- Precise calculations
- Database queries
- Web searches
- External API calls
- File operations
- Sending messages
- Other application-specific actions

The agent decides:

1. Whether a tool is needed
2. Which tool to use
3. What arguments to pass
4. How to use the tool result in its final answer

---

## Key Concepts

### 1. The `@tool` Decorator

LangChain's `@tool` decorator converts a normal Python function into an agent-compatible tool.

The decorator uses:

- The function name as the tool name
- The docstring as the tool description
- Type hints to create the argument schema
- The function body as the tool implementation

Example:

```python
@tool
def square_root(x: float) -> float:
    """Calculate the square root of a non-negative number."""
    return x ** 0.5
```

---

### 2. Tool Names and Descriptions

The model reads the tool name and description when deciding whether to call a tool.

A clear description helps the model select the correct tool.

Less useful:

```python
"""Do math."""
```

Better:

```python
"""Calculate the square root of a non-negative number."""
```

Tool descriptions should explain:

- What the tool does
- When it should be used
- What its arguments mean
- Any important limitations

---

### 3. Type Hints

Type hints define the expected tool arguments.

```python
def square_root(x: float) -> float:
```

In this example:

- `x` must be a floating-point number
- The tool returns a floating-point number

Type hints help LangChain build the schema that the model uses when generating tool-call arguments.

---

### 4. Ways to Define a Tool

The notebook demonstrates three equivalent styles.

#### Default name and docstring description

```python
@tool
def square_root(x: float) -> float:
    """Calculate the square root of a number."""
    return x ** 0.5
```

#### Custom tool name

```python
@tool("square_root")
def calculate_value(x: float) -> float:
    """Calculate the square root of a number."""
    return x ** 0.5
```

#### Custom name and description

```python
@tool(
    "square_root",
    description="Calculate the square root of a number."
)
def calculate_value(x: float) -> float:
    return x ** 0.5
```

The third style is useful when the Python function name or developer-oriented docstring should differ from what the model sees.

---

### 5. Test a Tool Directly

Tools should be tested independently before attaching them to an agent.

```python
square_root.invoke({"x": 467})
```

Direct testing:

- Does not require an LLM call
- Does not consume model tokens
- Helps isolate tool implementation errors
- Confirms that the input schema works

---

### 6. Attach Tools to an Agent

Tools are passed to an agent with the `tools` parameter.

```python
agent = create_agent(
    model="gpt-5-nano",
    tools=[square_root]
)
```

Multiple tools can be provided:

```python
agent = create_agent(
    model="gpt-5-nano",
    tools=[square_root, square_number]
)
```

The agent can then select the appropriate tool based on the user request and the tool descriptions.

---

### 7. Tool-Calling Message Flow

When a tool is used, the message history generally follows this pattern:

```text
[0] HumanMessage
    User's question

[1] AIMessage
    Model's tool-call decision

[2] ToolMessage
    Result returned by the Python tool

[3] AIMessage
    Final natural-language answer
```

The intermediate `AIMessage` may have empty text content because it contains a `tool_calls` entry instead of a normal answer.

---

### 8. Get the Final Answer

Use the last message to retrieve the final response:

```python
response["messages"][-1].content
```

Using `[-1]` is safer than using a fixed index because the number of messages can vary depending on whether the model calls a tool.

---

### 9. Inspect Tool Calls

The tool-call decision can be inspected with:

```python
response["messages"][1].tool_calls
```

This can show:

- The selected tool name
- The arguments passed to the tool
- The tool-call identifier
- The tool-call type

This is useful for debugging incorrect tool selection or arguments.

---

## Combined Example

The following single example combines the main concepts from the notebook:

- Loading environment variables
- Defining a tool
- Providing a clear description
- Using type hints
- Testing the tool directly
- Attaching it to an agent
- Allowing the agent to call it
- Printing the final response
- Inspecting the full message history
- Inspecting the tool-call decision

```python
from dotenv import load_dotenv
from pprint import pprint

from langchain.agents import create_agent
from langchain.messages import HumanMessage
from langchain.tools import tool


# ------------------------------------------------------------
# 1. Load environment variables
# ------------------------------------------------------------

# Loads values such as OPENAI_API_KEY from a .env file.
load_dotenv()


# ------------------------------------------------------------
# 2. Define a tool
# ------------------------------------------------------------

@tool(
    "square_root",
    description=(
        "Calculate the exact square root of a non-negative number. "
        "Use this tool whenever the user asks for a square root."
    )
)
def calculate_square_root(x: float) -> float:
    """
    Return the square root of a non-negative number.
    """

    if x < 0:
        raise ValueError("The square root input must be non-negative.")

    return x ** 0.5


# ------------------------------------------------------------
# 3. Test the tool directly
# ------------------------------------------------------------

# This calls the Python function directly.
# No LLM call is made.
direct_result = calculate_square_root.invoke({"x": 467})

print("=== DIRECT TOOL RESULT ===")
print(direct_result)


# ------------------------------------------------------------
# 4. Create an agent with the tool
# ------------------------------------------------------------

agent = create_agent(
    model="gpt-5-nano",
    tools=[calculate_square_root],
    system_prompt="""
    You are an arithmetic assistant.

    Use the available tools whenever a calculation requires an exact result.
    Explain the final answer clearly after the tool returns its result.
    """
)


# ------------------------------------------------------------
# 5. Ask a question that requires the tool
# ------------------------------------------------------------

question = HumanMessage(
    content="What is the square root of 467?"
)

response = agent.invoke(
    {"messages": [question]}
)


# ------------------------------------------------------------
# 6. Print the final answer
# ------------------------------------------------------------

print("\n=== FINAL AGENT ANSWER ===")
print(response["messages"][-1].content)


# ------------------------------------------------------------
# 7. Inspect the complete message history
# ------------------------------------------------------------

print("\n=== MESSAGE HISTORY ===")

for index, message in enumerate(response["messages"]):
    print(f"\n--- Message {index} ---")
    print(f"Type: {type(message).__name__}")
    pprint(message)


# ------------------------------------------------------------
# 8. Inspect the tool-call decision
# ------------------------------------------------------------

print("\n=== TOOL CALLS ===")

for message in response["messages"]:
    if hasattr(message, "tool_calls") and message.tool_calls:
        pprint(message.tool_calls)
```

---

## Expected Message Flow

For a question that requires the square-root tool, the response normally contains messages similar to:

```text
HumanMessage
    "What is the square root of 467?"

AIMessage
    tool_calls=[
        {
            "name": "square_root",
            "args": {"x": 467},
            ...
        }
    ]

ToolMessage
    "21.587..."

AIMessage
    "The square root of 467 is approximately 21.59."
```
---

## Microsoft Agent Framework Example

The following C# example demonstrates the equivalent tool-calling workflow using the Microsoft Agent Framework.

The tool is defined as a standard C# method, converted into an `AIFunction`, registered in `AgentOptions`, and then provided to a `ChatAgent`.

```csharp
using Microsoft.Extensions.AI;
using Microsoft.Agents.Extensions;

// 1. Define your tool as a standard C# method with a description.
[AIFunction("Gets the current weather for a given city")]
public string GetWeather(string city)
{
    // Replace with an actual weather API call if needed.
    if (city.Equals("Seattle", StringComparison.OrdinalIgnoreCase))
    {
        return "The weather in Seattle is rainy and 55°F.";
    }

    return $"The weather in {city} is sunny and 72°F.";
}

// 2. Wrap the method into an AIFunction tool.
var weatherTool = AIFunctionFactory.Create(GetWeather);

// 3. Register the tool when initializing the agent.
var agentOptions = new AgentOptions
{
    Tools = [weatherTool]
};

// 4. Create the agent inside the application process.
var agent = new ChatAgent(
    "WeatherAgent",
    agentOptions
);
```

### Microsoft Agent Framework Concepts

| Concept | Description |
|---|---|
| Tool method | A normal C# method that performs an action or retrieves data |
| `AIFunction` | Provides a description of the method for the AI model |
| `AIFunctionFactory.Create(...)` | Converts the C# method into a callable AI tool |
| `AgentOptions.Tools` | Registers tools that the agent can use |
| `ChatAgent` | Represents the configured agent running inside the application |

### Microsoft Tool-Calling Flow

```text
User request
    ↓
ChatAgent receives the request
    ↓
The model decides whether to call GetWeather
    ↓
AIFunction invokes the C# method
    ↓
The weather result is returned to the model
    ↓
The agent produces the final response
```

### Important Notes

- The `GetWeather` method must be defined inside a valid C# class or application type.
- The method description helps the model understand when the tool should be used.
- The method can be replaced with a real weather-service API call.
- Tool registration is performed through `AgentOptions`.
- The tool executes inside the application's process.

---

## Microsoft Agent Framework Packages

Add the following packages or namespaces to the package section:

| Package or namespace | Usage |
|---|---|
| `Microsoft.Extensions.AI` | Provides AI abstractions such as `AIFunction` and `AIFunctionFactory` |
| `Microsoft.Agents.Extensions` | Provides Microsoft Agent Framework extensions such as agent configuration and `ChatAgent` |
| `System` | Provides standard C# functionality such as `StringComparison` |

Example imports:

```csharp
using Microsoft.Extensions.AI;
using Microsoft.Agents.Extensions;
```

---

## LangChain and Microsoft Agent Framework Comparison

| Capability | LangChain | Microsoft Agent Framework |
|---|---|---|
| Tool definition | Python function with `@tool` | C# method with `AIFunction` |
| Tool description | Function docstring or decorator description | Attribute or function metadata |
| Tool wrapper | LangChain tool object | `AIFunctionFactory.Create(...)` |
| Tool registration | `tools=[...]` in `create_agent` | `AgentOptions.Tools` |
| Agent creation | `create_agent(...)` | `new ChatAgent(...)` |
| Tool invocation | `tool.invoke({...})` | Method invocation through `AIFunction` |
| Debugging | Inspect `messages` and `tool_calls` | Inspect agent/tool execution and application logs |
---

## Key Packages Used

| Package or module | Usage |
|---|---|
| `langchain` | Agent and tool functionality |
| `langchain.agents` | Provides `create_agent` |
| `langchain.tools` | Provides the `@tool` decorator |
| `langchain.messages` | Provides `HumanMessage` and other message types |
| `python-dotenv` | Loads environment variables from `.env` |
| `pprint` | Python standard library module for readable debugging output |

### Imports from the notebook

```python
from dotenv import load_dotenv
from langchain.agents import create_agent
from langchain.messages import HumanMessage
from langchain.tools import tool
from pprint import pprint
```

---

## Installation

```bash
pip install -U langchain python-dotenv
```

`pprint` is included with Python and does not need to be installed separately.

The notebook metadata uses Python 3.12.

---

## Recommended Tool-Design Rules

1. Give each tool one clear responsibility.
2. Use descriptive tool names.
3. Write docstrings for the model, not only for developers.
4. Add type hints to every tool parameter.
5. Test tools directly before attaching them to an agent.
6. Return simple values such as strings, numbers, or dictionaries.
7. Inspect `tool_calls` when the agent behaves unexpectedly.
8. Use `messages[-1]` to retrieve the final answer.
9. Validate tool inputs inside the function.
10. Avoid exposing powerful side-effect tools without appropriate safeguards.

---

## Summary

| Concept | Main takeaway |
|---|---|
| Tools | Python functions that extend an agent's capabilities |
| `@tool` | Converts a Python function into a LangChain tool |
| Docstrings | Help the model understand when to use a tool |
| Type hints | Define the tool's input schema |
| `.invoke()` | Executes a tool directly or invokes an agent |
| `tools=[...]` | Gives an agent access to one or more tools |
| `messages[-1]` | Retrieves the final agent answer |
| `.tool_calls` | Shows which tool was selected and which arguments were passed |
| Message history | Helps debug the agent's tool-calling process |
