+++
date = '2026-05-29T15:05:01+08:00'
draft = false
title = '200 行代码，彻底讲清楚 AI Agent 的底层运行机制'
+++

当我们使用 Claude Code、Cursor 等 AI 编程工具时，只需要输入一段自然语言指令，AI 就能够理解项目结构、阅读代码、修改文件，甚至自主完成多步骤的开发任务。

整个过程看起来颇具智能：AI 能够理解复杂需求，自主判断下一步应该做什么，并调用不同工具完成任务。

但这些能力背后的技术原理，真的有那么复杂吗？

Mihail Eric 在文章《The Emperor Has No Clothes》中给出了一个有趣的答案：**一个具备基本能力的 Coding Agent，其核心实现只需要约 200 行 Python 代码。**

当然，这并不意味着 Claude Code 这样的成熟产品只由 200 行代码构成，而是说明其底层最核心的 Agent 执行机制，并没有想象中那么神秘。

本文将从零开始，拆解一个简化版 Coding Agent 的实现过程，理解 AI Agent 如何思考、调用工具、执行任务，以及如何通过循环机制完成复杂工作。

## 一、AI Agent 的核心：LLM + Tools + Execution Loop

要理解 AI Agent，首先需要建立一个基本认知：

**AI Agent 本质上是一个具备工具调用能力、能够持续与外部环境交互的大语言模型应用。**

传统的大语言模型主要通过输入上下文生成文本。用户提出问题，模型生成答案，交互通常在一次响应后结束。

而 Agent 不同。

它不仅需要理解用户的需求，还需要根据任务目标判断是否需要调用外部工具，并根据工具执行结果决定下一步行动。

以创建一个 Python 文件为例，整个过程如下：

1. 用户提出需求：创建一个包含 Hello World 函数的 Python 文件。
2. LLM 分析需求，判断需要调用文件编辑工具。
3. LLM 生成结构化工具调用指令。
4. Agent 执行程序接收指令，在本地文件系统中创建文件。
5. 程序将执行结果返回给 LLM。
6. LLM 根据执行结果判断任务是否完成，并向用户反馈。

整个过程形成一个持续交互的闭环：

**用户指令 → LLM 决策 → 工具调用 → 外部执行 → 结果反馈 → LLM 再决策。**

这里有一个非常重要的技术细节：

LLM 本身并不会直接访问本地文件系统，也不会亲自执行文件创建、修改等操作。它只是生成工具调用请求，真正执行这些操作的是 Agent 所在的程序。

这也是 Agent 与普通 LLM 对话应用之间最重要的区别之一。

## 二、构建 Coding Agent，只需要三种基础工具

一个 Coding Agent 要具备基本的代码操作能力，需要让 LLM 能够感知和修改外部项目环境。

本例使用三种最基础的工具：

### 1. Read File：文件读取

让 Agent 能够读取指定文件的内容。

例如，用户要求修改某个 Python 文件中的函数，Agent 首先需要读取文件，了解现有代码结构，避免在不了解上下文的情况下直接修改。

```python
def read_file_tool(filename):
    full_path = resolve_abs_path(filename)

    with open(full_path, "r") as f:
        content = f.read()

    return {
        "file_path": str(full_path),
        "content": content
    }
```

该工具接收文件路径，读取文件内容，并将文件路径和内容封装成结构化结果返回。

### 2. List Files：文件列表

让 Agent 能够了解项目目录结构。

例如，当用户要求分析整个项目时，Agent 需要知道项目中有哪些文件、目录，以及代码组织方式。

```python
def list_files_tool(path):
    full_path = resolve_abs_path(path)

    all_files = []

    for item in full_path.iterdir():
        all_files.append({
            "filename": item.name,
            "type": "file" if item.is_file() else "dir"
        })

    return {
        "path": str(full_path),
        "files": all_files
    }
```

通过文件列表工具，LLM 可以逐步建立对项目结构的认知，并决定接下来应该读取哪些文件。

### 3. Edit File：文件编辑

让 Agent 能够真正修改代码，本例实现两种操作模式：

- 创建文件：当 old_str 为空时，将 new_str 写入文件。
- 修改文件：找到 old_str 对应的内容，并使用 new_str 替换。

```python
def edit_file_tool(path, old_str, new_str):
    full_path = resolve_abs_path(path)

    if old_str == "":
        full_path.write_text(new_str, encoding="utf-8")
        return {
            "path": str(full_path),
            "action": "created_file"
        }

    original = full_path.read_text(encoding="utf-8")

    if old_str not in original:
        return {
            "path": str(full_path),
            "action": "old_str not found"
        }

    edited = original.replace(old_str, new_str, 1)
    full_path.write_text(edited, encoding="utf-8")

    return {
        "path": str(full_path),
        "action": "edited"
    }
```

至此，Agent 已经具备了最基本的代码环境交互能力：

- 通过读取文件获取代码上下文；
- 通过遍历目录了解项目结构；
- 通过编辑文件执行代码修改。

这三种工具构成了一个最小化 Coding Agent 的基础能力。

## 三、Tool Registry：让 LLM 知道自己能做什么

有了工具，还需要解决一个问题：

LLM 如何知道当前有哪些工具可以使用？每个工具需要什么参数？应该在什么情况下调用？

这就需要建立一个工具注册机制（Tool Registry）。

```python
TOOL_REGISTRY = {
    "read_file": read_file_tool,
    "list_files": list_files_tool,
    "edit_file": edit_file_tool
}
```

通过字典维护工具名称与实际执行函数之间的映射关系。

但仅仅注册工具还不够，还需要将工具的名称、功能描述和参数结构提供给 LLM。

例子中采用 Python 的 inspect 模块，动态读取函数签名和 Docstring，生成工具描述信息。

```python
def get_tool_str_representation(tool_name):
    tool = TOOL_REGISTRY[tool_name]

    return f"""
    Name: {tool_name}
    Description: {tool.__doc__}
    Signature: {inspect.signature(tool)}
    """
```

随后，将所有工具描述整合到 System Prompt 中。

这样，LLM 在处理用户任务时，就能够根据工具说明判断：

- 当前有哪些工具可用？
- 每个工具可以完成什么任务？
- 调用工具时需要提供哪些参数？

这里体现了 Agent 开发中的一个重要设计思想：

**工具不仅需要被程序注册，还需要被模型理解。**

工具的功能描述、参数定义和使用约束，实际上构成了 LLM 与外部执行环境之间的接口契约。

## 四、Tool Calling：LLM 如何驱动外部工具执行？

接下来是整个 Agent 架构中非常关键的环节：工具调用。

在简化实现中，通过 Prompt 约定了一种工具调用格式：

```text
tool: TOOL_NAME({"参数": "参数值"})
```

例如，用户要求创建一个 hello.py 文件，LLM 可能返回：

```text
tool: edit_file({
    "path": "hello.py",
    "old_str": "",
    "new_str": "print('Hello World')"
})
```

需要注意，这里展示的是简化后的文本协议。实际代码中要求工具调用使用单行 JSON 格式。

Agent 程序通过解析模型输出，识别是否存在工具调用请求。

```python
def extract_tool_invocations(text):
    invocations = []

    for raw_line in text.splitlines():
        line = raw_line.strip()

        if not line.startswith("tool:"):
            continue

        try:
            after = line[len("tool:"):].strip()
            name, rest = after.split("(", 1)

            if not rest.endswith(")"):
                continue

            json_str = rest[:-1].strip()
            args = json.loads(json_str)

            invocations.append((name.strip(), args))

        except Exception:
            continue

    return invocations
```

解析完成后，程序根据工具名称从 TOOL_REGISTRY 中找到对应的 Python 函数，并执行调用。

整个过程可以理解为：

1. LLM 生成工具调用指令。
2. Agent 解析工具名称和参数。
3. Agent 找到对应的执行函数。
4. 执行函数并获取结果。
5. 将执行结果返回给 LLM。

由此，LLM 的语言生成能力就与外部系统的实际操作能力连接起来了。

在成熟的 Agent 框架中，这一机制通常通过模型原生的 Function Calling 或 Tool Calling 接口实现，而不是依赖文本解析。这些接口能够提供更标准化的工具定义和结构化调用参数。

## 五、Agent Loop：让 AI 从单次响应走向自主执行

如果说前面的工具定义、工具注册和工具调用解决了 Agent 如何使用工具的问题，那么 Agent Loop 则解决了另一个关键问题：

**如何让 AI 持续执行任务，而不是调用一次工具就结束？**

这是 Agent 与普通工具调用应用之间最重要的区别之一。

通过一个循环机制实现了这一能力。

其核心逻辑是：

```python
while True:

    assistant_response = execute_llm_call(conversation)

    tool_invocations = extract_tool_invocations(
        assistant_response
    )

    if not tool_invocations:
        print(assistant_response)
        break

    for name, args in tool_invocations:

        tool = TOOL_REGISTRY[name]

        result = tool(**args)

        conversation.append({
            "role": "user",
            "content": f"tool_result({json.dumps(result)})"
        })
```

这段代码展示了 Agent 的核心运行机制（[获取完整代码](https://github.com/AIScientist-101/simple_aiagent_with_200_code)）。

当用户提出任务后，Agent 首先调用 LLM。

如果 LLM 返回普通文本，说明当前没有需要执行的工具调用，Agent 就将结果返回给用户。

如果 LLM 返回工具调用请求，Agent 就执行对应工具，并将工具执行结果追加到对话上下文中。

随后，再次调用 LLM。

LLM 根据新的上下文判断下一步行动，可能继续调用其他工具，也可能直接给出最终答案。

整个过程持续循环，直到 LLM 不再请求调用工具。

例如：

用户提出：

「修改 hello.py，在原有代码基础上增加一个两个数字相乘的函数。」

Agent 的执行过程可能是：

**第一步：读取文件**

LLM 判断需要先了解文件内容，于是调用 read_file。

**第二步：获取执行结果**

Agent 读取 hello.py，将文件内容返回给 LLM。

**第三步：分析并修改**

LLM 根据文件内容生成修改指令，调用 edit_file。

**第四步：执行修改**

Agent 执行文件编辑操作，并将修改结果返回给 LLM。

**第五步：完成任务**

LLM 根据工具执行结果，向用户反馈代码已经修改完成。

可以看到，Agent 并不是提前写死了所有操作步骤，而是根据每一步的执行结果，动态决定下一步应该做什么。

这就是 Agent 的核心运行逻辑：**感知环境、制定行动、执行工具、获取反馈，并根据反馈持续调整行动。**

## 六、200 行代码实现的 Agent，与生产级 Agent 有什么区别？

通过上述实现，我们已经能够构建一个具备基本文件操作能力的 Coding Agent。

但这距离 Claude Code 这样的生产级产品，还有相当大的差距。生产级 Agent 需要在基础执行闭环之外，增加更多工程能力。

主要包括以下几个方面：

**1. 更完善的异常处理机制**

工具调用可能失败，例如文件不存在、权限不足、参数错误等。

生产级 Agent 需要识别异常，将错误信息反馈给 LLM，并允许模型根据错误信息重新制定执行策略。

**2. 更丰富的工具体系**

除了读取、列出和编辑文件，还需要支持：

- Shell / Bash：执行终端命令；
- Grep：搜索代码和文本；
- Web Search：检索外部信息；
- Code Execution：执行代码并获取运行结果；
- Git：管理代码版本和变更。

工具体系越丰富，Agent 能够与外部环境交互的范围就越广。

**3. 上下文管理能力**

随着任务不断推进，对话上下文会持续增长。

当项目规模较大、执行步骤较多时，Agent 需要具备上下文压缩、历史信息总结、关键信息保留等能力，避免上下文过长影响模型的推理效果。

**4. 流式响应机制**

通过 Streaming 机制，模型可以逐步返回生成内容和执行状态，减少用户等待时间，提升交互体验。

**5. 安全控制与人工审批机制**

Agent 一旦拥有文件修改、命令执行等能力，就可能对外部环境产生真实影响。

因此，需要建立权限管理、操作确认、危险命令拦截、文件访问范围限制等机制。

特别是涉及删除文件、修改系统配置、执行高风险命令等操作时，不能完全依赖 LLM 自主判断。

这些工程能力，才是一个简单 Agent 原型走向生产级 Agent 系统时真正需要解决的问题。

## 七、从 200 行代码理解 AI Agent 的本质

回顾整个实现过程，可以发现，Agent 的核心架构实际上非常清晰：

- LLM：负责理解任务、分析上下文和制定行动决策；
- Tools：提供与外部环境交互的具体能力；
- Tool Registry：管理工具及其描述信息；
- Execution Loop：协调模型决策、工具执行和结果反馈；
- Context：维护任务执行过程中不断积累的信息。

这些模块共同构成了一个最基本的 Agent 执行系统。

从这个角度来看，Agent 并不是一个全新的、难以理解的神秘技术，而是在传统 LLM 应用之上，引入了工具调用、外部执行和循环决策机制。

当然，真正构建一个可靠、稳定、可扩展的生产级 Agent，远不只是实现一个循环那么简单。工具权限、上下文管理、任务规划、执行状态管理、异常恢复、可观测性和安全治理，都是必须认真考虑的工程问题。

但理解这个最小化实现，能够帮助我们抓住 Agent 技术最核心的部分。

**Agent 的关键，不在于模型有多么神奇，而在于如何通过工程架构，将 LLM 的推理与决策能力转化为真实世界中的执行能力。**

这或许也是理解 Claude Code、OpenAI Codex 以及其他 AI Agent 产品最好的切入点。
