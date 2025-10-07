# ReAct 技术详解：从原理到实践

## 目录
1. [ReAct 概述](#1-react-概述)
2. [ReAct 核心原理](#2-react-核心原理)
3. [ReAct 的判断标志](#3-react-的判断标志)
4. [ReAct 工作流程](#4-react-工作流程)
5. [ReAct 在 AgentScope 中的实现](#5-react-在-agentscope-中的实现)
6. [如何开发 ReAct 智能体](#6-如何开发-react-智能体)
7. [ReAct vs 其他方法](#7-react-vs-其他方法)
8. [最佳实践与案例](#8-最佳实践与案例)

---

## 1. ReAct 概述

### 1.1 什么是 ReAct？

**ReAct** = **Rea**soning（推理） + **Act**ing（行动）

ReAct 是一种由 Princeton 大学和 Google 研究团队于 2022 年提出的智能体架构模式，发表在 ICLR 2023 会议上。它的核心思想是：**让大语言模型（LLM）交替生成推理轨迹（Thought）和执行行动（Action），通过观察（Observation）反馈来持续优化决策过程**。

### 1.2 论文信息

- **论文标题**：ReAct: Synergizing Reasoning and Acting in Language Models
- **作者**：Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik Narasimhan, Yuan Cao
- **发表会议**：ICLR 2023
- **ArXiv 编号**：arXiv:2210.03629
- **项目主页**：https://react-lm.github.io/
- **代码仓库**：https://github.com/ysymyth/ReAct

### 1.3 为什么需要 ReAct？

**传统方法的局限性**：

1. **纯推理方法（如 Chain-of-Thought）**：
   - 容易产生幻觉（Hallucination）
   - 无法获取外部最新信息
   - 错误会传播和累积

2. **纯行动方法（如 Tool Use）**：
   - 缺乏规划能力
   - 无法处理复杂的多步骤任务
   - 难以从错误中恢复

**ReAct 的优势**：
- ✅ **减少幻觉**：通过外部工具获取真实信息
- ✅ **可解释性强**：推理过程透明可见
- ✅ **错误恢复能力**：观察结果后可调整策略
- ✅ **任务分解能力**：将复杂任务拆解为多个步骤

---

## 2. ReAct 核心原理

### 2.1 核心循环：Think → Act → Observe

ReAct 的核心是一个循环过程：

```
┌─────────────────────────────────────────────┐
│                                             │
│  1. Thought (思考)                          │
│     ↓                                       │
│  2. Action (行动)                           │
│     ↓                                       │
│  3. Observation (观察)                      │
│     ↓                                       │
│  4. 返回步骤1，直到任务完成                   │
│                                             │
└─────────────────────────────────────────────┘
```

### 2.2 三个关键要素

#### 1️⃣ **Thought（思考/推理）**
- **定义**：智能体对当前状态的分析和下一步计划
- **作用**：
  - 分析已有信息
  - 规划后续行动
  - 追踪任务进度
  - 处理异常情况
- **示例**：
  ```
  Thought: 用户想知道2024年奥运会金牌榜，我需要搜索最新信息。
  ```

#### 2️⃣ **Action（行动）**
- **定义**：智能体执行的具体操作，通常是调用工具函数
- **作用**：
  - 获取外部信息（搜索、API 调用）
  - 执行计算任务（代码执行、数学计算）
  - 操作环境（文件读写、命令执行）
- **示例**：
  ```
  Action: search("2024 Olympics medal count")
  ```

#### 3️⃣ **Observation（观察）**
- **定义**：行动执行后的反馈结果
- **作用**：
  - 提供新的信息
  - 验证行动结果
  - 触发下一轮推理
- **示例**：
  ```
  Observation: 美国以40金38银42铜位居金牌榜首位...
  ```

### 2.3 完整示例

**用户问题**：2024年奥运会哪个国家金牌最多？

**ReAct 执行过程**：

```
Thought 1: 我需要搜索2024年奥运会的金牌榜信息。
Action 1: search("2024 Olympics gold medal ranking")
Observation 1: 2024年巴黎奥运会，美国以40枚金牌位居第一...

Thought 2: 我已经找到了答案，美国是金牌最多的国家。
Action 2: generate_response("根据2024年巴黎奥运会的结果，美国以40枚金牌位居金牌榜首位。")
Observation 2: [任务完成]
```

---

## 3. ReAct 的判断标志

### 3.1 核心判断标准 ⭐⭐⭐⭐⭐

**一个智能体使用了 ReAct，当且仅当满足以下所有条件**：

#### ✅ 标志 1：具有明确的推理-行动循环（Reasoning-Acting Loop）

**特征**：
- 智能体在执行任务时，**交替进行推理和行动**
- 推理步骤生成思考过程（Thought）
- 行动步骤调用工具或执行操作（Action）
- 每次行动后都有观察反馈（Observation）

**代码特征**：
```python
# ReAct 的核心循环结构
for iteration in range(max_iters):
    # 1. 推理阶段（Reasoning）
    thought_and_action = await self._reasoning()

    # 2. 行动阶段（Acting）
    observation = await self._acting(thought_and_action)

    # 3. 检查是否完成任务
    if task_completed(observation):
        break
```

#### ✅ 标志 2：拥有工具调用能力（Tool Calling）

**特征**：
- 智能体必须能够调用外部工具或函数
- 工具返回的结果会影响下一步的推理
- 工具集合可以包括：
  - 信息检索工具（搜索、知识库查询）
  - 计算工具（代码执行、数学计算）
  - 环境交互工具（文件操作、命令执行）

**代码特征**：
```python
# 注册工具函数
toolkit = Toolkit()
toolkit.register_tool_function(search_tool)
toolkit.register_tool_function(calculator)
toolkit.register_tool_function(code_executor)

# 在 Acting 阶段调用工具
tool_result = await toolkit.call_tool_function(action)
```

#### ✅ 标志 3：具有观察-反馈机制（Observation-Feedback）

**特征**：
- 每次工具调用后，结果会被反馈给智能体
- 智能体根据观察结果调整下一步策略
- 观察结果会被添加到对话历史中

**代码特征**：
```python
# 执行工具并获取观察结果
observation = await execute_tool(action)

# 将观察结果记录到记忆中
await self.memory.add(observation_message)

# 下一轮推理时，观察结果会影响决策
next_thought = await self._reasoning()  # 包含之前的观察结果
```

#### ✅ 标志 4：支持多轮迭代（Iterative Process）

**特征**：
- 智能体可以执行多轮推理-行动循环
- 每轮都基于之前的观察结果进行决策
- 直到任务完成或达到最大迭代次数

**代码特征**：
```python
# 设置最大迭代次数
max_iters = 10

# 多轮循环
for i in range(max_iters):
    reasoning_msg = await self._reasoning()
    acting_msg = await self._acting(reasoning_msg)

    if is_final_answer(acting_msg):
        break
```

#### ✅ 标志 5：具有任务终止机制（Task Completion）

**特征**：
- 智能体能够判断任务是否完成
- 通常有一个特殊的"完成"函数（如 `generate_response`）
- 调用完成函数后退出循环

**代码特征**：
```python
def generate_response(self, response: str) -> ToolResponse:
    """生成最终回复并结束任务"""
    return ToolResponse(
        content=response,
        metadata={"task_completed": True}
    )

# 在循环中检测任务完成
if tool_name == "generate_response":
    return final_message  # 退出循环
```

### 3.2 AgentScope 中的 ReAct 标志

在 AgentScope 框架中，判断是否使用了 ReAct 的**具体代码特征**：

#### 1. **继承 `ReActAgent` 或 `ReActAgentBase` 类**

```python
from agentscope.agent import ReActAgent

# 明确使用 ReAct 架构
agent = ReActAgent(
    name="Assistant",
    sys_prompt="You are a helpful assistant.",
    model=model,
    formatter=formatter,
    toolkit=toolkit,
)
```

#### 2. **实现了 `_reasoning()` 和 `_acting()` 方法**

```python
class MyReActAgent(ReActAgentBase):
    async def _reasoning(self) -> Msg:
        """推理阶段：生成思考和行动"""
        # 调用 LLM 生成推理和工具调用
        prompt = self.prepare_prompt()
        response = await self.model(prompt, tools=self.toolkit.get_json_schemas())
        return response

    async def _acting(self, tool_call: ToolUseBlock) -> Msg:
        """行动阶段：执行工具并返回观察结果"""
        # 执行工具调用
        result = await self.toolkit.call_tool_function(tool_call)
        return result
```

#### 3. **在 `reply()` 方法中实现循环**

```python
async def reply(self, msg: Msg) -> Msg:
    """ReAct 的核心 reply 方法"""
    await self.memory.add(msg)

    # ReAct 循环
    for _ in range(self.max_iters):
        # 1. 推理
        reasoning_msg = await self._reasoning()

        # 2. 行动
        tool_calls = reasoning_msg.get_content_blocks("tool_use")
        for tool_call in tool_calls:
            acting_result = await self._acting(tool_call)

            # 3. 检查是否完成
            if is_final_response(acting_result):
                return acting_result

    # 超过最大迭代次数
    return self._summarizing()
```

#### 4. **注册了工具函数并在推理中使用**

```python
# 创建工具集
toolkit = Toolkit()
toolkit.register_tool_function(search_web)
toolkit.register_tool_function(execute_python_code)

# 在模型调用时传递工具模式
response = await model(
    prompt,
    tools=toolkit.get_json_schemas()  # 关键：传递工具定义
)
```

#### 5. **具有 `generate_response` 或类似的终止函数**

```python
def generate_response(self, response: str) -> ToolResponse:
    """生成最终响应（ReAct 的终止标志）"""
    response_msg = Msg(self.name, response, "assistant")
    return ToolResponse(
        content="Task completed",
        metadata={
            "success": True,
            "response_msg": response_msg
        }
    )

# 注册为工具
toolkit.register_tool_function(self.generate_response)
```

### 3.3 非 ReAct 的对照

**以下情况不算使用 ReAct**：

❌ **单次工具调用**：
```python
# 这不是 ReAct，只是简单的函数调用
result = search_web("AgentScope")
print(result)
```

❌ **没有推理过程的工具链**：
```python
# 这不是 ReAct，只是顺序执行工具
result1 = tool1()
result2 = tool2(result1)
result3 = tool3(result2)
```

❌ **纯推理没有行动**：
```python
# 这不是 ReAct，只是 Chain-of-Thought
thought1 = "首先我需要..."
thought2 = "然后我应该..."
answer = "所以答案是..."
```

❌ **没有观察反馈的行动**：
```python
# 这不是 ReAct，工具结果没有被用于后续推理
for action in action_list:
    execute(action)  # 结果被丢弃
```

### 3.4 快速检查清单

**判断一个智能体是否使用了 ReAct，检查以下几点**：

| 检查项 | 说明 | 是否必须 |
|--------|------|----------|
| 🔄 循环结构 | 是否有多轮 Reasoning-Acting 循环？ | ✅ 必须 |
| 🤔 推理阶段 | 是否生成思考过程（Thought）？ | ✅ 必须 |
| 🔧 工具调用 | 是否调用外部工具（Action）？ | ✅ 必须 |
| 👁️ 观察反馈 | 工具结果是否反馈给智能体（Observation）？ | ✅ 必须 |
| 🔁 迭代决策 | 是否基于观察结果调整策略？ | ✅ 必须 |
| 🛑 终止机制 | 是否有明确的任务完成判断？ | ✅ 必须 |
| 📝 记忆管理 | 是否记录历史对话和观察？ | ⚠️ 推荐 |
| 🔀 并行工具 | 是否支持并行调用多个工具？ | ⭕ 可选 |

---

## 4. ReAct 工作流程

### 4.1 完整流程图

```
┌──────────────────────────────────────────────────────────────┐
│                      用户输入问题                              │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                  Step 1: 初始化                               │
│  - 加载系统提示词                                              │
│  - 准备工具集合                                                │
│  - 初始化记忆                                                  │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────────┐
        │   Step 2: 推理阶段（Reasoning）     │
        │                                    │
        │  1. 构建提示词：                    │
        │     - 系统提示词                    │
        │     - 对话历史                      │
        │     - 可用工具列表                  │
        │                                    │
        │  2. 调用 LLM 生成：                │
        │     - Thought（思考）               │
        │     - Action（工具调用）            │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │   Step 3: 行动阶段（Acting）        │
        │                                    │
        │  1. 解析工具调用请求                │
        │  2. 执行工具函数                    │
        │  3. 获取执行结果                    │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │   Step 4: 观察阶段（Observation）   │
        │                                    │
        │  1. 记录工具执行结果                │
        │  2. 更新对话历史                    │
        │  3. 分析是否达成目标                │
        └────────────┬───────────────────────┘
                     │
                     ▼
                ┌─────────┐
                │ 任务完成？│
                └────┬────┘
                     │
           ┌─────────┴─────────┐
           │                   │
          是                   否
           │                   │
           ▼                   ▼
    ┌──────────────┐    ┌──────────────┐
    │ 返回最终答案  │    │ 返回 Step 2  │
    └──────────────┘    └──────────────┘
                              │
                              └──→ 继续循环（最多 max_iters 次）
```

### 4.2 详细步骤说明

#### **Step 1: 初始化阶段**

```python
# 1. 创建智能体
agent = ReActAgent(
    name="Assistant",
    sys_prompt="You are a helpful AI assistant.",
    model=ChatModel(),
    formatter=Formatter(),
    toolkit=Toolkit(),
    max_iters=10  # 最大循环次数
)

# 2. 用户输入
user_msg = Msg("user", "2024年奥运会金牌榜第一是谁？", "user")
```

#### **Step 2: 推理阶段（Reasoning）**

```python
async def _reasoning(self) -> Msg:
    # 1. 准备提示词
    prompt = [
        {"role": "system", "content": self.sys_prompt},
        *self.memory.get_memory(),  # 历史对话
    ]

    # 2. 调用 LLM，传递工具定义
    response = await self.model(
        prompt,
        tools=self.toolkit.get_json_schemas()
    )

    # 3. 返回包含 Thought 和 Action 的消息
    # response.content 可能是：
    # [
    #     {"type": "text", "text": "我需要搜索2024奥运会信息"},
    #     {"type": "tool_use", "name": "search", "input": {"query": "2024 Olympics medal"}}
    # ]
    return response
```

**LLM 生成示例**：
```
Thought: 用户想知道2024年奥运会金牌榜第一名。我需要搜索最新的奥运会信息。
Action: search(query="2024 Olympics gold medal ranking")
```

#### **Step 3: 行动阶段（Acting）**

```python
async def _acting(self, tool_call: ToolUseBlock) -> Msg:
    # 1. 解析工具调用
    tool_name = tool_call["name"]  # "search"
    tool_input = tool_call["input"]  # {"query": "2024 Olympics..."}

    # 2. 执行工具
    result = await self.toolkit.call_tool_function(tool_call)

    # 3. 构建观察结果消息
    observation_msg = Msg(
        "system",
        [ToolResultBlock(
            type="tool_result",
            id=tool_call["id"],
            name=tool_name,
            output=result.content
        )],
        "system"
    )

    return observation_msg
```

**工具执行示例**：
```
Action: search(query="2024 Olympics gold medal ranking")
Observation: 美国以40金38银42铜位居2024年巴黎奥运会金牌榜首位...
```

#### **Step 4: 观察阶段（Observation）**

```python
# 1. 记录观察结果
await self.memory.add(observation_msg)

# 2. 检查是否为最终答案
if tool_call["name"] == "generate_response":
    # 任务完成
    return final_response
else:
    # 继续下一轮推理
    continue
```

**决策流程**：
```python
# 循环继续
for i in range(max_iters):
    # 推理
    reasoning_msg = await self._reasoning()

    # 行动
    for tool_call in reasoning_msg.get_tool_calls():
        observation = await self._acting(tool_call)

        # 检查是否完成
        if is_final_answer(observation):
            return observation
```

#### **Step 5: 生成最终答案**

```python
# 第二轮推理（基于观察结果）
Thought: 我已经获得了答案，美国是金牌榜第一。
Action: generate_response(
    response="根据2024年巴黎奥运会的结果，美国以40枚金牌位居金牌榜首位。"
)

# 返回最终答案并结束循环
```

### 4.3 完整执行示例

**任务**：计算 (123 + 456) * 2 的结果

**ReAct 执行轨迹**：

```
用户: 计算 (123 + 456) * 2

--- 第 1 轮 ---
Thought 1: 我需要先计算 123 + 456，然后将结果乘以 2。
Action 1: calculator(expression="123 + 456")
Observation 1: 579

--- 第 2 轮 ---
Thought 2: 现在我需要将 579 乘以 2。
Action 2: calculator(expression="579 * 2")
Observation 2: 1158

--- 第 3 轮 ---
Thought 3: 我已经得到了最终答案 1158。
Action 3: generate_response(response="(123 + 456) * 2 = 1158")
Observation 3: [任务完成]

智能体: (123 + 456) * 2 = 1158
```

---

## 5. ReAct 在 AgentScope 中的实现

### 5.1 核心类结构

```python
# 基类：定义 ReAct 的抽象接口
class ReActAgentBase(AgentBase):
    @abstractmethod
    async def _reasoning(self, *args, **kwargs) -> Any:
        """推理阶段：生成思考和行动计划"""
        pass

    @abstractmethod
    async def _acting(self, *args, **kwargs) -> Any:
        """行动阶段：执行工具并获取观察结果"""
        pass

# 完整实现：AgentScope 的 ReAct 智能体
class ReActAgent(ReActAgentBase):
    def __init__(
        self,
        name: str,
        sys_prompt: str,
        model: ChatModelBase,
        formatter: FormatterBase,
        toolkit: Toolkit | None = None,
        memory: MemoryBase | None = None,
        max_iters: int = 10,
        parallel_tool_calls: bool = False,
        # ... 其他参数
    ):
        """初始化 ReAct 智能体"""
        self.name = name
        self.sys_prompt = sys_prompt
        self.model = model
        self.formatter = formatter
        self.toolkit = toolkit or Toolkit()
        self.memory = memory or InMemoryMemory()
        self.max_iters = max_iters
        self.parallel_tool_calls = parallel_tool_calls

        # 注册终止函数
        self.toolkit.register_tool_function(self.generate_response)

    async def reply(self, msg: Msg) -> Msg:
        """ReAct 的核心回复逻辑"""
        await self.memory.add(msg)

        # ReAct 循环
        for _ in range(self.max_iters):
            # 1. 推理
            reasoning_msg = await self._reasoning()

            # 2. 行动
            tool_calls = reasoning_msg.get_content_blocks("tool_use")
            for tool_call in tool_calls:
                acting_result = await self._acting(tool_call)

                # 3. 检查是否完成
                if acting_result and acting_result.metadata.get("task_completed"):
                    return acting_result

        # 超过最大迭代次数，生成总结
        return await self._summarizing()

    async def _reasoning(self) -> Msg:
        """推理阶段实现"""
        # 1. 准备提示词
        prompt = await self.formatter.format([
            Msg("system", self.sys_prompt, "system"),
            *await self.memory.get_memory(),
        ])

        # 2. 调用 LLM，传递工具定义
        response = await self.model(
            prompt,
            tools=self.toolkit.get_json_schemas()
        )

        # 3. 处理流式输出或普通输出
        if self.model.stream:
            msg = Msg(self.name, [], "assistant")
            async for chunk in response:
                msg.content = chunk.content
                await self.print(msg, False)
            await self.print(msg, True)
        else:
            msg = Msg(self.name, response.content, "assistant")
            await self.print(msg, True)

        # 4. 记录到记忆
        await self.memory.add(msg)
        return msg

    async def _acting(self, tool_call: ToolUseBlock) -> Msg | None:
        """行动阶段实现"""
        # 1. 执行工具
        tool_result = await self.toolkit.call_tool_function(tool_call)

        # 2. 构建观察消息
        observation_msg = Msg(
            "system",
            [ToolResultBlock(
                type="tool_result",
                id=tool_call["id"],
                name=tool_call["name"],
                output=tool_result.content
            )],
            "system"
        )

        # 3. 打印和记录
        await self.print(observation_msg, True)
        await self.memory.add(observation_msg)

        # 4. 检查是否为终止函数
        if tool_call["name"] == "generate_response":
            if tool_result.metadata.get("success"):
                return tool_result.metadata.get("response_msg")

        return None

    def generate_response(self, response: str, **kwargs) -> ToolResponse:
        """生成最终回复（终止函数）"""
        response_msg = Msg(self.name, response, "assistant")

        return ToolResponse(
            content=[TextBlock(type="text", text="Task completed")],
            metadata={
                "success": True,
                "response_msg": response_msg,
                "task_completed": True
            },
            is_last=True
        )
```

### 5.2 关键特性

#### 1. **Hook 机制**

AgentScope 的 ReAct 支持在推理和行动的前后注册钩子函数：

```python
# 支持的 Hook 类型
supported_hooks = [
    "pre_reasoning",   # 推理前
    "post_reasoning",  # 推理后
    "pre_acting",      # 行动前
    "post_acting",     # 行动后
]

# 注册 Hook
agent.register_instance_hook(
    "pre_reasoning",
    "my_hook",
    lambda self, kwargs: print("开始推理...")
)
```

#### 2. **并行工具调用**

```python
# 启用并行工具调用
agent = ReActAgent(
    name="Assistant",
    parallel_tool_calls=True,  # 并行执行多个工具
    # ...
)

# 内部实现
if self.parallel_tool_calls:
    # 并行执行
    results = await asyncio.gather(*[
        self._acting(tool_call)
        for tool_call in tool_calls
    ])
else:
    # 顺序执行
    results = [await self._acting(tc) for tc in tool_calls]
```

#### 3. **长期记忆与知识库集成**

```python
agent = ReActAgent(
    name="Assistant",
    # 长期记忆
    long_term_memory=LongTermMemory(),
    long_term_memory_mode="both",  # agent_control | static_control | both
    # 知识库
    knowledge=[KnowledgeBase1(), KnowledgeBase2()],
    enable_rewrite_query=True,  # 重写查询以提高检索效果
    # ...
)
```

#### 4. **计划笔记本（Plan Notebook）**

```python
plan_notebook = PlanNotebook()

agent = ReActAgent(
    name="Assistant",
    plan_notebook=plan_notebook,  # 支持复杂任务分解
    # ...
)

# 智能体可以调用计划工具
# - create_plan: 创建计划
# - update_subtask_state: 更新子任务状态
# - finish_subtask: 完成子任务
# - finish_plan: 完成整个计划
```

### 5.3 AgentScope ReAct 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      ReActAgent                              │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              核心组件                               │    │
│  │                                                      │    │
│  │  - Model（语言模型）                                 │    │
│  │  - Formatter（格式化器）                             │    │
│  │  - Toolkit（工具集）                                 │    │
│  │  - Memory（短期记忆）                                │    │
│  │  - LongTermMemory（长期记忆，可选）                   │    │
│  │  - Knowledge（知识库，可选）                          │    │
│  │  - PlanNotebook（计划笔记本，可选）                   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              ReAct 循环（最多 max_iters 次）         │    │
│  │                                                      │    │
│  │  1. _reasoning():                                    │    │
│  │     ├─ 构建提示词（系统提示 + 历史 + Hint）          │    │
│  │     ├─ 调用 LLM（传递工具定义）                      │    │
│  │     └─ 返回 Thought + Action                        │    │
│  │                                                      │    │
│  │  2. _acting():                                       │    │
│  │     ├─ 解析工具调用                                  │    │
│  │     ├─ 执行工具函数                                  │    │
│  │     ├─ 获取 Observation                             │    │
│  │     └─ 检查是否完成任务                              │    │
│  │                                                      │    │
│  │  3. 循环控制:                                        │    │
│  │     ├─ 如果是 generate_response → 返回答案          │    │
│  │     ├─ 如果未完成 → 继续下一轮                       │    │
│  │     └─ 达到 max_iters → 生成总结                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Hook 系统                               │    │
│  │                                                      │    │
│  │  - pre_reasoning hooks                               │    │
│  │  - post_reasoning hooks                              │    │
│  │  - pre_acting hooks                                  │    │
│  │  - post_acting hooks                                 │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 如何开发 ReAct 智能体

### 6.1 基础版 ReAct 智能体

**需求**：创建一个能够搜索和计算的 ReAct 智能体

#### Step 1: 准备工具函数

```python
from agentscope.tool import Toolkit

# 1. 定义工具函数
def search_web(query: str) -> str:
    """在网上搜索信息

    Args:
        query: 搜索关键词

    Returns:
        搜索结果
    """
    # 实际实现中调用搜索 API
    return f"搜索结果：{query} 的相关信息..."

def calculator(expression: str) -> str:
    """执行数学计算

    Args:
        expression: 数学表达式

    Returns:
        计算结果
    """
    try:
        result = eval(expression)
        return f"计算结果：{result}"
    except Exception as e:
        return f"计算错误：{str(e)}"

# 2. 注册工具
toolkit = Toolkit()
toolkit.register_tool_function(search_web)
toolkit.register_tool_function(calculator)
```

#### Step 2: 创建 ReAct 智能体

```python
from agentscope.agent import ReActAgent
from agentscope.model import DashScopeChatModel
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory

# 创建模型
model = DashScopeChatModel(
    api_key="your_api_key",
    model_name="qwen-max"
)

# 创建 ReAct 智能体
agent = ReActAgent(
    name="Assistant",
    sys_prompt=(
        "You are a helpful AI assistant. "
        "You can search the web and perform calculations. "
        "Think step by step and use tools when needed."
    ),
    model=model,
    formatter=DashScopeChatFormatter(),
    toolkit=toolkit,
    memory=InMemoryMemory(),
    max_iters=10,  # 最多 10 轮循环
)
```

#### Step 3: 运行智能体

```python
import asyncio

async def main():
    # 测试问题
    user_input = "搜索2024年奥运会金牌榜，并计算前三名的金牌总数"

    # 调用智能体
    response = await agent(Msg("user", user_input, "user"))

    print(f"智能体回复：{response.content}")

asyncio.run(main())
```

**执行过程**：
```
Thought 1: 我需要先搜索2024年奥运会金牌榜信息。
Action 1: search_web(query="2024奥运会金牌榜")
Observation 1: 搜索结果：美国40金，中国40金，日本20金...

Thought 2: 现在我需要计算前三名的金牌总数：40 + 40 + 20。
Action 2: calculator(expression="40 + 40 + 20")
Observation 2: 计算结果：100

Thought 3: 我已经得到了答案。
Action 3: generate_response(
    response="根据搜索结果，2024年奥运会金牌榜前三名是美国（40金）、中国（40金）和日本（20金），总共100枚金牌。"
)

智能体回复：根据搜索结果，2024年奥运会金牌榜前三名是美国（40金）、中国（40金）和日本（20金），总共100枚金牌。
```

### 6.2 进阶版：带知识库的 ReAct 智能体

#### Step 1: 准备知识库

```python
from agentscope.rag import KnowledgeBase, InMemoryKnowledgeBase
from agentscope.rag import Document

# 创建知识库
knowledge = InMemoryKnowledgeBase(
    embedding_model="text-embedding-v2"
)

# 添加文档
documents = [
    Document(content="AgentScope 是一个多智能体平台..."),
    Document(content="ReAct 是推理和行动结合的方法..."),
    Document(content="2024年巴黎奥运会在法国举办..."),
]

await knowledge.add_documents(documents)
```

#### Step 2: 创建带 RAG 的 ReAct 智能体

```python
agent = ReActAgent(
    name="RAG_Assistant",
    sys_prompt="You are a helpful assistant with access to knowledge base.",
    model=model,
    formatter=formatter,
    toolkit=toolkit,
    knowledge=knowledge,  # 添加知识库
    enable_rewrite_query=True,  # 启用查询重写
    print_hint_msg=True,  # 打印提示信息
)
```

**工作流程**：
```
用户输入：什么是 AgentScope？

1. 查询重写：
   原始查询："什么是 AgentScope？"
   重写后："AgentScope 定义 功能 特点"

2. 知识检索：
   从知识库中检索到：
   - "AgentScope 是一个多智能体平台..."
   - "AgentScope 支持 ReAct 智能体..."

3. ReAct 推理：
   Thought: 我从知识库中找到了相关信息，可以直接回答。
   Action: generate_response(
       response="AgentScope 是一个多智能体平台，支持..."
   )
```

### 6.3 高级版：带计划能力的 ReAct 智能体

#### Step 1: 创建计划笔记本

```python
from agentscope.plan import PlanNotebook

plan_notebook = PlanNotebook(
    max_subtasks=10,  # 最多 10 个子任务
)
```

#### Step 2: 创建支持计划的 ReAct 智能体

```python
agent = ReActAgent(
    name="Planner_Assistant",
    sys_prompt=(
        "You are an AI assistant capable of breaking down complex tasks. "
        "Use planning tools to decompose tasks into subtasks."
    ),
    model=model,
    formatter=formatter,
    toolkit=toolkit,
    plan_notebook=plan_notebook,  # 添加计划笔记本
    enable_meta_tool=True,  # 启用元工具（动态工具管理）
)
```

#### Step 3: 执行复杂任务

```python
user_input = "帮我开发一个简单的待办事项应用"

response = await agent(Msg("user", user_input, "user"))
```

**执行过程**：
```
Thought 1: 这是一个复杂任务，我需要创建一个计划。
Action 1: create_plan(
    name="Todo App Development",
    description="开发一个待办事项应用",
    subtasks=[
        SubTask(name="设计数据模型", description="定义任务的数据结构"),
        SubTask(name="实现后端API", description="创建 CRUD 接口"),
        SubTask(name="开发前端界面", description="构建用户界面"),
        SubTask(name="测试和部署", description="测试并部署应用"),
    ]
)
Observation 1: 计划创建成功

Thought 2: 现在开始执行第一个子任务。
Action 2: update_subtask_state(subtask_idx=0, state="in_progress")
Observation 2: 子任务 0 已标记为进行中

Thought 3: 我需要设计数据模型...
Action 3: execute_python_code(
    code="""
    class Task:
        def __init__(self, title, description, status):
            self.title = title
            self.description = description
            self.status = status
    """
)
Observation 3: 代码执行成功

Thought 4: 数据模型已完成。
Action 4: finish_subtask(
    subtask_idx=0,
    outcome="已定义 Task 类，包含 title、description 和 status 字段"
)
Observation 4: 子任务 0 已完成，子任务 1 已激活

# 继续执行其他子任务...
```

### 6.4 自定义 ReAct 智能体

如果需要更灵活的控制，可以继承 `ReActAgentBase`：

```python
from agentscope.agent import ReActAgentBase
from agentscope.message import Msg

class MyReActAgent(ReActAgentBase):
    def __init__(self, name: str, model, toolkit):
        super().__init__()
        self.name = name
        self.model = model
        self.toolkit = toolkit
        self.memory = []
        self.max_iters = 5

    async def _reasoning(self) -> Msg:
        """自定义推理逻辑"""
        # 1. 构建提示词
        prompt = self._build_prompt()

        # 2. 调用模型
        response = await self.model(
            prompt,
            tools=self.toolkit.get_json_schemas()
        )

        # 3. 解析并返回
        return Msg(self.name, response.content, "assistant")

    async def _acting(self, tool_call) -> Msg:
        """自定义行动逻辑"""
        # 1. 执行工具
        result = await self.toolkit.call_tool_function(tool_call)

        # 2. 自定义处理逻辑
        if tool_call["name"] == "special_tool":
            # 特殊处理
            result = self._process_special_result(result)

        # 3. 返回观察结果
        return Msg("system", result.content, "system")

    async def reply(self, msg: Msg) -> Msg:
        """自定义 ReAct 循环"""
        self.memory.append(msg)

        for i in range(self.max_iters):
            # 推理
            reasoning_msg = await self._reasoning()
            self.memory.append(reasoning_msg)

            # 行动
            tool_calls = self._extract_tool_calls(reasoning_msg)
            for tc in tool_calls:
                observation = await self._acting(tc)
                self.memory.append(observation)

                # 自定义终止条件
                if self._should_stop(observation):
                    return self._generate_final_response()

        return Msg(self.name, "任务未完成", "assistant")

    def _build_prompt(self):
        """构建提示词"""
        return [
            {"role": "system", "content": "You are a helpful assistant."},
            *[{"role": m.role, "content": m.content} for m in self.memory]
        ]

    def _extract_tool_calls(self, msg: Msg):
        """提取工具调用"""
        return msg.get_content_blocks("tool_use")

    def _should_stop(self, observation: Msg):
        """判断是否停止"""
        return "任务完成" in observation.content

    def _generate_final_response(self):
        """生成最终回复"""
        return Msg(self.name, "任务已完成", "assistant")
```

---

## 7. ReAct vs 其他方法

### 7.1 方法对比

| 特性 | ReAct | Chain-of-Thought (CoT) | Tool Use | Function Calling |
|------|-------|------------------------|----------|------------------|
| **推理能力** | ✅ 强 | ✅ 强 | ❌ 弱 | ❌ 弱 |
| **工具调用** | ✅ 支持 | ❌ 不支持 | ✅ 支持 | ✅ 支持 |
| **外部信息** | ✅ 可获取 | ❌ 仅依赖参数知识 | ✅ 可获取 | ✅ 可获取 |
| **多步骤任务** | ✅ 擅长 | ⚠️ 有限 | ⚠️ 需要预定义 | ❌ 不擅长 |
| **可解释性** | ✅ 强（有推理过程） | ✅ 强 | ❌ 弱 | ❌ 弱 |
| **错误恢复** | ✅ 支持 | ❌ 不支持 | ⚠️ 有限 | ❌ 不支持 |
| **适用场景** | 复杂任务、需要工具 | 推理密集型任务 | 简单工具调用 | API 调用 |

### 7.2 详细对比

#### **1. ReAct vs Chain-of-Thought (CoT)**

**Chain-of-Thought**：
```
问题：2024年奥运会金牌榜第一是谁？

CoT 推理：
1. 2024年奥运会在巴黎举行
2. 通常美国和中国在金牌榜上排名靠前
3. 根据历史数据，美国可能是第一
4. 答案：美国（可能不准确，因为是基于猜测）
```

**ReAct**：
```
问题：2024年奥运会金牌榜第一是谁？

ReAct 执行：
Thought: 我需要搜索实际的金牌榜数据。
Action: search("2024 Olympics gold medal ranking")
Observation: 美国和中国并列第一，各40枚金牌

Thought: 我找到了准确答案。
Action: generate_response("2024年奥运会金牌榜上，美国和中国并列第一，各获得40枚金牌。")
```

**优势对比**：
- ✅ ReAct 获取真实数据，避免幻觉
- ✅ CoT 不需要工具，更简单
- ✅ ReAct 适合需要最新信息的任务
- ✅ CoT 适合纯推理任务

#### **2. ReAct vs Function Calling**

**Function Calling**（单次调用）：
```python
# 模型决定调用哪个函数
response = model.call(
    "帮我搜索 AgentScope",
    functions=[search_web, calculator]
)

# 执行函数
result = search_web(response.function_args["query"])

# 返回结果（无进一步推理）
return result
```

**ReAct**（多轮循环）：
```python
# 第1轮
Thought: 需要搜索 AgentScope
Action: search_web("AgentScope")
Observation: AgentScope 是一个多智能体平台...

# 第2轮
Thought: 我还想了解更多细节
Action: search_web("AgentScope features")
Observation: 支持分布式部署、ReAct 智能体...

# 第3轮
Thought: 信息已足够
Action: generate_response("AgentScope 是...")
```

**优势对比**：
- ✅ ReAct 支持多步骤推理和工具调用
- ✅ Function Calling 更简单，适合单步任务
- ✅ ReAct 有错误恢复能力
- ✅ Function Calling 延迟更低

### 7.3 选择指南

**选择 ReAct 的场景**：
- ✅ 任务需要多步骤推理和工具调用
- ✅ 需要从错误中恢复
- ✅ 任务复杂度高，需要动态规划
- ✅ 需要可解释的决策过程

**选择 CoT 的场景**：
- ✅ 纯推理任务（数学、逻辑）
- ✅ 不需要外部信息
- ✅ 追求简单性和低延迟

**选择 Function Calling 的场景**：
- ✅ 简单的工具调用
- ✅ 明确的单步任务
- ✅ 低延迟要求

**选择 Tool Use 的场景**：
- ✅ 预定义的工具流程
- ✅ 不需要动态推理
- ✅ 批量处理任务

---

## 8. 最佳实践与案例

### 8.1 最佳实践

#### 1. **设计高质量的系统提示词**

**推荐**：
```python
sys_prompt = """You are a helpful AI assistant with access to various tools.

Guidelines:
1. Think step by step before taking actions
2. Use tools when you need external information or computation
3. Always verify the results before providing final answers
4. If you encounter errors, try alternative approaches
5. Provide clear explanations with your responses

Available tools: {tool_list}
"""
```

**避免**：
```python
# 过于简单
sys_prompt = "You are a helpful assistant."

# 过于复杂
sys_prompt = "You are a super intelligent AI with..."  # 500 字
```

#### 2. **合理设置最大迭代次数**

```python
# 根据任务复杂度设置
simple_agent = ReActAgent(
    max_iters=3,  # 简单任务
)

complex_agent = ReActAgent(
    max_iters=15,  # 复杂任务
)

# 监控实际使用情况
@agent.register_hook("post_reply")
async def monitor_iterations(self, kwargs, output):
    iterations = getattr(self, '_iteration_count', 0)
    if iterations >= self.max_iters:
        logger.warning(f"Reached max iterations: {iterations}")
```

#### 3. **工具函数设计原则**

**良好的工具函数**：
```python
def search_arxiv(
    query: str,
    max_results: int = 5,
    sort_by: str = "relevance"
) -> str:
    """在 arXiv 上搜索学术论文

    Args:
        query: 搜索关键词，例如 "transformer architecture"
        max_results: 返回结果数量，默认 5 篇
        sort_by: 排序方式，可选 "relevance" 或 "date"

    Returns:
        JSON 格式的搜索结果，包含标题、作者、摘要等
    """
    # 实现...
    return json.dumps(results)
```

**特点**：
- ✅ 明确的参数说明
- ✅ 合理的默认值
- ✅ 结构化的返回值
- ✅ 完善的文档字符串

#### 4. **错误处理和重试机制**

```python
async def _acting(self, tool_call):
    """带重试机制的 Acting"""
    max_retries = 3

    for attempt in range(max_retries):
        try:
            result = await self.toolkit.call_tool_function(tool_call)
            return result
        except Exception as e:
            if attempt < max_retries - 1:
                # 添加错误提示
                error_hint = Msg(
                    "system",
                    f"Tool execution failed: {str(e)}. Please try again.",
                    "system"
                )
                await self.memory.add(error_hint)
            else:
                raise e
```

#### 5. **使用 Hook 进行监控和调试**

```python
# 监控推理时间
async def measure_reasoning_time(self, kwargs):
    self._reasoning_start = time.time()

async def log_reasoning_time(self, kwargs, output):
    elapsed = time.time() - self._reasoning_start
    logger.info(f"Reasoning took {elapsed:.2f}s")

agent.register_instance_hook("pre_reasoning", "timer_start", measure_reasoning_time)
agent.register_instance_hook("post_reasoning", "timer_end", log_reasoning_time)

# 记录工具调用
async def log_tool_calls(self, kwargs):
    tool_call = kwargs.get("tool_call")
    logger.info(f"Calling tool: {tool_call['name']} with {tool_call['input']}")

agent.register_instance_hook("pre_acting", "log_tools", log_tool_calls)
```

### 8.2 实战案例

#### 案例 1：数据分析助手

**需求**：分析 CSV 文件并生成报告

```python
import pandas as pd
from agentscope.tool import Toolkit

# 1. 定义工具
def read_csv(file_path: str) -> str:
    """读取 CSV 文件"""
    df = pd.read_csv(file_path)
    return df.head(10).to_string()

def analyze_data(file_path: str, operation: str) -> str:
    """分析数据

    Args:
        file_path: CSV 文件路径
        operation: 分析操作，可选 "summary", "correlation", "missing"
    """
    df = pd.read_csv(file_path)

    if operation == "summary":
        return df.describe().to_string()
    elif operation == "correlation":
        return df.corr().to_string()
    elif operation == "missing":
        return df.isnull().sum().to_string()

def plot_data(file_path: str, x_col: str, y_col: str) -> str:
    """绘制图表"""
    df = pd.read_csv(file_path)
    df.plot(x=x_col, y=y_col)
    plt.savefig("plot.png")
    return "图表已保存到 plot.png"

# 2. 创建智能体
toolkit = Toolkit()
toolkit.register_tool_function(read_csv)
toolkit.register_tool_function(analyze_data)
toolkit.register_tool_function(plot_data)

agent = ReActAgent(
    name="DataAnalyst",
    sys_prompt=(
        "You are a data analysis assistant. "
        "Help users analyze CSV files and generate insights."
    ),
    model=model,
    formatter=formatter,
    toolkit=toolkit,
)

# 3. 使用
user_query = "分析 sales_data.csv 文件，找出销售额和月份的关系"

# 执行过程：
# Thought 1: 先读取文件了解数据结构
# Action 1: read_csv("sales_data.csv")
# Observation 1: [显示前10行数据]

# Thought 2: 生成统计摘要
# Action 2: analyze_data("sales_data.csv", "summary")
# Observation 2: [统计摘要]

# Thought 3: 绘制销售额与月份的关系图
# Action 3: plot_data("sales_data.csv", "month", "sales")
# Observation 3: 图表已保存

# Thought 4: 生成分析报告
# Action 4: generate_response("根据分析...")
```

#### 案例 2：代码助手

**需求**：帮助用户编写、运行和调试代码

```python
from agentscope.tool import execute_python_code, view_text_file

# 1. 扩展工具
def write_code_to_file(file_path: str, code: str) -> str:
    """将代码写入文件"""
    with open(file_path, 'w') as f:
        f.write(code)
    return f"代码已写入 {file_path}"

def run_tests(test_file: str) -> str:
    """运行测试"""
    import subprocess
    result = subprocess.run(
        ["pytest", test_file],
        capture_output=True,
        text=True
    )
    return result.stdout + result.stderr

# 2. 创建智能体
toolkit = Toolkit()
toolkit.register_tool_function(execute_python_code)
toolkit.register_tool_function(write_code_to_file)
toolkit.register_tool_function(view_text_file)
toolkit.register_tool_function(run_tests)

agent = ReActAgent(
    name="CodeAssistant",
    sys_prompt=(
        "You are a coding assistant. Help users write, test, and debug code. "
        "Always test code before finalizing."
    ),
    model=model,
    formatter=formatter,
    toolkit=toolkit,
)

# 3. 使用
user_query = "写一个函数计算斐波那契数列，并测试它"

# 执行过程：
# Thought 1: 先写斐波那契函数
# Action 1: execute_python_code("""
# def fibonacci(n):
#     if n <= 1:
#         return n
#     return fibonacci(n-1) + fibonacci(n-2)
# print([fibonacci(i) for i in range(10)])
# """)
# Observation 1: [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Thought 2: 代码工作正常，保存到文件
# Action 2: write_code_to_file("fibonacci.py", "def fibonacci...")
# Observation 2: 代码已写入 fibonacci.py

# Thought 3: 编写测试
# Action 3: write_code_to_file("test_fibonacci.py", "import pytest...")
# Observation 3: 测试已写入

# Thought 4: 运行测试
# Action 4: run_tests("test_fibonacci.py")
# Observation 4: All tests passed

# Thought 5: 完成任务
# Action 5: generate_response("已完成斐波那契函数...")
```

#### 案例 3：研究助手

**需求**：帮助用户进行文献调研

```python
# 1. 定义研究工具
def search_arxiv(query: str, max_results: int = 5) -> str:
    """搜索 arXiv 论文"""
    # 调用 arxiv API
    papers = arxiv_api.search(query, max_results=max_results)
    return json.dumps([{
        "title": p.title,
        "authors": p.authors,
        "abstract": p.summary[:200] + "...",
        "url": p.pdf_url
    } for p in papers])

def download_paper(url: str, save_path: str) -> str:
    """下载论文 PDF"""
    # 下载逻辑
    return f"论文已下载到 {save_path}"

def extract_key_points(paper_path: str) -> str:
    """提取论文要点"""
    # 使用 LLM 提取关键信息
    text = extract_text_from_pdf(paper_path)
    summary = llm.summarize(text)
    return summary

# 2. 创建知识库
knowledge = InMemoryKnowledgeBase()

# 3. 创建研究助手
toolkit = Toolkit()
toolkit.register_tool_function(search_arxiv)
toolkit.register_tool_function(download_paper)
toolkit.register_tool_function(extract_key_points)

agent = ReActAgent(
    name="ResearchAssistant",
    sys_prompt=(
        "You are a research assistant. Help users find, read, "
        "and summarize academic papers."
    ),
    model=model,
    formatter=formatter,
    toolkit=toolkit,
    knowledge=knowledge,
)

# 4. 使用
user_query = "调研关于 Transformer 注意力机制的最新进展"

# 执行过程：
# Thought 1: 搜索相关论文
# Action 1: search_arxiv("Transformer attention mechanism", 10)
# Observation 1: [返回10篇论文]

# Thought 2: 下载最相关的3篇
# Action 2: download_paper(url1, "paper1.pdf")
# Action 3: download_paper(url2, "paper2.pdf")
# Action 4: download_paper(url3, "paper3.pdf")

# Thought 3: 提取关键信息
# Action 5: extract_key_points("paper1.pdf")
# Observation 5: [论文1要点]
# Action 6: extract_key_points("paper2.pdf")
# ...

# Thought 4: 生成综述报告
# Action: generate_response("关于 Transformer 注意力机制的最新进展...")
```

### 8.3 性能优化技巧

#### 1. **并行工具调用**

```python
# 当多个工具调用相互独立时
agent = ReActAgent(
    parallel_tool_calls=True,  # 启用并行
)

# 例如：同时搜索多个数据源
# Thought: 我需要从多个来源获取信息
# Action 1: search_google("query")
# Action 2: search_arxiv("query")
# Action 3: search_wikipedia("query")
# 这三个调用会并行执行，节省时间
```

#### 2. **缓存工具结果**

```python
from functools import lru_cache

@lru_cache(maxsize=100)
def expensive_search(query: str) -> str:
    """带缓存的搜索"""
    # 实际搜索逻辑
    return search_api(query)

# 相同查询会直接返回缓存结果
```

#### 3. **流式输出**

```python
# 启用流式输出，提升响应速度
model = DashScopeChatModel(
    model_name="qwen-max",
    stream=True,  # 启用流式
)

agent = ReActAgent(
    model=model,
    # ...
)

# 用户可以看到实时的推理过程
```

---

## 9. 总结

### 9.1 核心要点

1. **ReAct 定义**：
   - ReAct = Reasoning（推理） + Acting（行动）
   - 交替进行思考和工具调用
   - 通过观察反馈优化决策

2. **判断标志**（全部满足才是 ReAct）：
   - ✅ 具有推理-行动循环
   - ✅ 支持工具调用
   - ✅ 具有观察-反馈机制
   - ✅ 支持多轮迭代
   - ✅ 具有任务终止机制

3. **核心流程**：
   ```
   循环 {
       1. Reasoning: 分析当前状态，规划行动
       2. Acting: 调用工具，执行操作
       3. Observation: 观察结果，更新认知
       4. 判断: 任务是否完成？
   }
   ```

4. **AgentScope 实现特点**：
   - 完整的 ReAct 基础设施
   - Hook 机制支持扩展
   - 集成 RAG、计划、长期记忆
   - 并行工具调用优化

### 9.2 学习路径

**入门**：
1. 理解 ReAct 核心概念
2. 使用 AgentScope 创建基础 ReAct 智能体
3. 实践简单的工具调用任务

**进阶**：
1. 设计复杂的工具集合
2. 集成知识库和长期记忆
3. 使用计划笔记本处理复杂任务

**高级**：
1. 自定义 ReAct 循环逻辑
2. 实现多智能体 ReAct 协作
3. 优化性能和错误处理

### 9.3 参考资源

1. **论文**：
   - ReAct: Synergizing Reasoning and Acting in Language Models (ICLR 2023)
   - https://arxiv.org/abs/2210.03629

2. **AgentScope 文档**：
   - 官方文档：https://agentscope.io
   - GitHub：https://github.com/modelscope/agentscope

3. **示例代码**：
   - AgentScope 示例目录：`examples/react_agent/`
   - 本文档配套代码示例

---

**最后总结**：ReAct 是一种强大的智能体架构模式，通过**推理-行动-观察**的循环，使 LLM 能够利用外部工具解决复杂任务。判断是否使用 ReAct 的核心标志是：**是否有明确的推理-行动循环、工具调用能力、观察反馈机制和多轮迭代过程**。在 AgentScope 中，通过 `ReActAgent` 类可以快速构建 ReAct 智能体，并通过丰富的扩展功能（RAG、计划、Hook）应对各种复杂场景。
