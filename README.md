# 🧠🤖Deep Agents

使用 LLM 在循环中调用工具是代理最简单的形式。
然而，这种架构可能会产生"浅层"代理，无法规划和执行更长期、更复杂的任务。
像 "Deep Research"、"Manus" 和 "Claude Code" 这样的应用通过实现四个要素的组合来克服这个限制：
**规划工具**、**子代理**、**文件系统访问**和**详细的提示**。

<img src="deep_agents.png" alt="deep agent" width="600"/>

`deepagents` 是一个 Python 包，它以通用的方式实现了这些功能，让您可以轻松地为您的应用创建深度代理。

**致谢：这个项目主要受到 Claude Code 的启发，最初主要是尝试了解什么让 Claude Code 具有通用性，并使其更加通用。**

## 安装

```bash
pip install deepagents
```

## 使用方法

（要运行下面的示例，需要先 `pip install tavily-python`）

```python
import os
from typing import Literal

from tavily import TavilyClient
from deepagents import create_deep_agent


# 用于研究的搜索工具
def internet_search(
    query: str,
    max_results: int = 5,
    topic: Literal["general", "news", "finance"] = "general",
    include_raw_content: bool = False,
):
    """运行网络搜索"""
    tavily_async_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])
    return tavily_async_client.search(
        query,
        max_results=max_results,
        include_raw_content=include_raw_content,
        topic=topic,
    )


# 引导代理成为专业研究员的提示前缀
research_instructions = """您是一位专业研究员。您的工作是进行深入研究，然后撰写精美的报告。

您可以使用一些工具。

## `internet_search`

使用这个工具为给定的查询运行互联网搜索。您可以指定结果数量、主题以及是否应包含原始内容。
"""

# 创建代理
agent = create_deep_agent(
    [internet_search],
    research_instructions,
)

# 调用代理
result = agent.invoke({"messages": [{"role": "user", "content": "什么是 langgraph？"}]})
```

查看 [examples/research/research_agent.py](examples/research/research_agent.py) 获取更复杂的示例。

使用 `create_deep_agent` 创建的代理只是一个 LangGraph 图 - 因此您可以像与任何 LangGraph 代理一样与其交互（流式传输、人机协作、记忆、studio）。

## 创建自定义深度代理

您可以向 `create_deep_agent` 传递三个参数来创建自己的自定义深度代理。

### `tools`（必需）

`create_deep_agent` 的第一个参数是 `tools`。
这应该是函数或 LangChain `@tool` 对象的列表。
代理（和任何子代理）将可以访问这些工具。

### `instructions`（必需）

`create_deep_agent` 的第二个参数是 `instructions`。
这将作为深度代理提示的一部分。
请注意，还有一个[内置系统提示](#内置提示)，所以这不是代理将看到的*全部*提示。

### `subagents`（可选）

`create_deep_agent` 的仅限关键字参数是 `subagents`。
这可用于指定此深度代理将访问的任何自定义子代理。
您可以在[这里](#子代理)阅读更多关于为什么要使用子代理的信息。

`subagents` 应该是字典列表，其中每个字典遵循此架构：

```python
class SubAgent(TypedDict):
    name: str
    description: str
    prompt: str
    tools: NotRequired[list[str]]
```

- **name**: 这是子代理的名称，以及主代理如何调用子代理
- **description**: 这是显示给主代理的子代理描述
- **prompt**: 这是用于子代理的提示
- **tools**: 这是子代理可以访问的工具列表。默认情况下将可以访问所有传入的工具以及所有内置工具。

使用示例：

```python
research_sub_agent = {
    "name": "research-agent",
    "description": "用于更深入地研究问题",
    "prompt": sub_research_prompt,
}
subagents = [research_subagent]
agent = create_deep_agent(
    tools,
    prompt,
    subagents=subagents
)
```

### `model`（可选）

默认情况下，`deepagents` 将使用 `"claude-sonnet-4-20250514"`。如果您想使用不同的模型，
可以传递一个 [LangChain 模型对象](https://python.langchain.com/docs/integrations/chat/)。

## 深度代理详情

以下组件内置于 `deepagents` 中，使其能够开箱即用地处理深度任务。

### 系统提示

`deepagents` 带有[内置系统提示](src/deepagents/prompts.py)。这是一个相对详细的提示，在很大程度上基于并受到[尝试](https://github.com/kn1026/cc/blob/main/claudecode.md)[复制](https://github.com/asgeirtj/system_prompts_leaks/blob/main/Anthropic/claude-code.md) Claude Code 系统提示的启发。它比 Claude Code 的系统提示更加通用。
这包含了如何使用内置规划工具、文件系统工具和子代理的详细说明。
请注意，此系统提示的部分内容[可以自定义](#instructions-必需)。

如果没有这个默认的系统提示 - 代理在执行任务时不会像现在这样成功。
提示对于创建"深度"代理的重要性不容小觑。

### 规划工具

`deepagents` 带有内置的规划工具。这个规划工具非常简单，基于 ClaudeCode 的 TodoWrite 工具。
这个工具实际上不做任何事情 - 它只是代理制定计划的一种方式，然后将其保存在上下文中以帮助保持正轨。

### 文件系统工具

`deepagents` 带有四个内置的文件系统工具：`ls`、`edit_file`、`read_file`、`write_file`。
这些实际上不使用文件系统 - 而是使用 LangGraph 的 State 对象模拟文件系统。
这意味着您可以轻松地在同一台机器上运行许多这些代理，而不必担心它们会编辑相同的底层文件。

目前"文件系统"只有一层深度（没有子目录）。

这些文件可以通过使用 LangGraph State 对象中的 `files` 键传入（也可以检索）。

```python
agent = create_deep_agent(...)

result = agent.invoke({
    "messages": ...,
    # 使用此键将文件传递给代理
    # "files": {"foo.txt": "foo", ...}
})

# 之后像这样访问任何文件
result["files"]
```

### 子代理

`deepagents` 内置了调用子代理的能力（基于 Claude Code）。
它始终可以访问 `general-purpose` 子代理 - 这是一个具有与主代理相同指令和它可以访问的所有工具的子代理。
您还可以指定[自定义子代理](#subagents-可选)，具有自己的指令和工具。

子代理对于["上下文隔离"](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html#context-quarantine)（帮助不污染主代理的整体上下文）以及自定义指令很有用。

## 路线图
[] 允许用户自定义完整的系统提示
[] 代码整洁性（类型提示、文档字符串、格式化）
[] 允许更健壮的虚拟文件系统
[] 创建基于此构建的深度编码代理示例
[] 对[深度研究代理](examples/research/research_agent.py)示例进行基准测试
[] 为工具添加人机协作支持
