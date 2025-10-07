# AgentScope 项目架构深度分析文档

> 本文档详细分析 AgentScope 多 Agent 框架的项目结构、核心模块、实现机制和调用关系

**文档版本**: 1.0
**项目版本**: AgentScope 1.0.4
**分析日期**: 2025-10-06
**项目地址**: https://github.com/agentscope-ai/agentscope

---

## 目录

- [1. 项目概览](#1-项目概览)
- [2. 核心架构](#2-核心架构)
- [3. 模块详细分析](#3-模块详细分析)
  - [3.1 Agent 模块](#31-agent-模块)
  - [3.2 Model 模块](#32-model-模块)
  - [3.3 Message 模块](#33-message-模块)
  - [3.4 Tool 模块](#34-tool-模块)
  - [3.5 Memory 模块](#35-memory-模块)
  - [3.6 Formatter 模块](#36-formatter-模块)
  - [3.7 Pipeline 模块](#37-pipeline-模块)
  - [3.8 Plan 模块](#38-plan-模块)
  - [3.9 RAG 模块](#39-rag-模块)
  - [3.10 MCP 模块](#310-mcp-模块)
  - [3.11 其他核心模块](#311-其他核心模块)
- [4. 模块间依赖与调用关系](#4-模块间依赖与调用关系)
- [5. 关键设计模式](#5-关键设计模式)
- [6. 执行流程分析](#6-执行流程分析)
- [7. 扩展开发指南](#7-扩展开发指南)
- [8. 最佳实践](#8-最佳实践)

---

## 1. 项目概览

### 1.1 项目简介

**AgentScope** 是由阿里巴巴通义实验室 SysML 团队开发的开源多 Agent 框架，专注于为 LLM 应用构建提供面向 Agent 的编程范式。

**核心特性**:
- ✨ **透明性优先**: 所有操作对开发者可见可控，无深度封装
- ⚡ **异步优先**: 全面支持异步执行和并发
- 🔧 **高度模块化**: LEGO 式组件设计
- 🤖 **多 Agent 导向**: 显式消息传递和工作流编排
- 🛠️ **模型无关**: 支持多种 LLM 提供商
- 🎯 **实时控制**: 原生支持实时中断和自定义处理

### 1.2 技术栈

**核心依赖**:
```python
# 必需依赖
aioitertools       # 异步迭代工具
anthropic          # Anthropic Claude API
dashscope          # 阿里云通义千问 API
openai             # OpenAI API
mcp                # Model Context Protocol
opentelemetry-*    # 分布式追踪
tiktoken           # Token 计数
```

**可选依赖**:
```python
# 模型相关
ollama, google-genai, transformers

# RAG 相关
pypdf, nltk, qdrant-client

# 长期记忆
mem0ai

# 评估
ray
```

### 1.3 项目目录结构

```
agentscope/
├── src/agentscope/          # 核心源码
│   ├── agent/              # Agent 实现
│   ├── model/              # 模型接口
│   ├── tool/               # 工具管理
│   ├── formatter/          # 提示词格式化
│   ├── memory/             # 记忆系统
│   ├── message/            # 消息系统
│   ├── pipeline/           # 工作流编排
│   ├── plan/               # 规划系统
│   ├── rag/                # RAG 模块
│   ├── mcp/                # MCP 客户端
│   ├── session/            # 会话管理
│   ├── embedding/          # 嵌入模型
│   ├── token/              # Token 处理
│   ├── evaluate/           # 评估框架
│   ├── tracing/            # 追踪系统
│   ├── hooks/              # 钩子系统
│   ├── module/             # 基础模块
│   ├── exception/          # 异常处理
│   ├── types/              # 类型定义
│   └── _utils/             # 工具函数
├── examples/               # 示例代码
├── tests/                  # 测试代码
├── docs/                   # 文档
└── setup.py               # 安装配置
```

### 1.4 初始化入口

**文件位置**: `src/agentscope/__init__.py`

**初始化函数**:
```python
def init(
    project: str | None = None,           # 项目名称
    name: str | None = None,              # 运行名称
    logging_path: str | None = None,      # 日志路径
    logging_level: str = "INFO",          # 日志级别
    studio_url: str | None = None,        # Studio URL
    tracing_url: str | None = None,       # 追踪端点 URL
) -> None:
    """初始化 AgentScope 库"""
```

**使用示例**:
```python
import agentscope

agentscope.init(
    project="my_project",
    name="run_001",
    logging_level="INFO",
    studio_url="http://localhost:5000"
)
```

---

## 2. 核心架构

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      AgentScope 核心架构                      │
└─────────────────────────────────────────────────────────────┘

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  UserAgent   │    │  ReActAgent  │    │ CustomAgent  │
│              │    │              │    │              │
│  用户交互     │◄──►│  ReAct循环   │◄──►│  自定义逻辑   │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
              ┌────────────▼────────────┐
              │     AgentBase          │
              │  - observe()           │
              │  - reply()             │
              │  - __call__()          │
              └────────────┬────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼─────┐    ┌──────▼──────┐    ┌─────▼──────┐
   │  Memory  │    │   Model     │    │  Toolkit   │
   │          │    │             │    │            │
   │  记忆管理 │    │  LLM 接口   │    │  工具管理   │
   └────┬─────┘    └──────┬──────┘    └─────┬──────┘
        │                 │                  │
   ┌────▼─────────┐  ┌────▼────────┐   ┌────▼─────────┐
   │ InMemory     │  │ DashScope   │   │ ToolFunction │
   │ LongTerm     │  │ OpenAI      │   │ MCPClient    │
   └──────────────┘  │ Anthropic   │   └──────────────┘
                     │ Gemini      │
                     │ Ollama      │
                     └─────────────┘

┌─────────────────────────────────────────────────────────────┐
│                        辅助模块                              │
├─────────────────────────────────────────────────────────────┤
│  Pipeline  │  Plan  │  RAG  │  Formatter  │  Tracing  │...  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心概念

#### 2.2.1 Agent（智能体）

**定义**: 能够感知环境、做出决策并采取行动的自主实体。

**核心特性**:
- 拥有独立的记忆
- 可以调用工具
- 支持异步执行
- 支持钩子扩展

**主要类型**:
1. **AgentBase**: 基础 Agent 类
2. **ReActAgent**: 实现 ReAct（Reasoning + Acting）模式
3. **UserAgent**: 用户交互 Agent

#### 2.2.2 Message（消息）

**定义**: Agent 之间通信的基本单位。

**结构**:
```python
class Msg:
    name: str                              # 发送者
    content: str | Sequence[ContentBlock]  # 内容
    role: Literal["user", "assistant", "system"]
    metadata: dict | None                  # 元数据
    timestamp: str                         # 时间戳
    invocation_id: str                     # 调用ID
```

**内容块类型**:
- `TextBlock`: 文本
- `ImageBlock`: 图片
- `AudioBlock`: 音频
- `VideoBlock`: 视频
- `ToolUseBlock`: 工具调用
- `ToolResultBlock`: 工具结果
- `ThinkingBlock`: 思考过程

#### 2.2.3 Tool（工具）

**定义**: Agent 可以调用的外部功能。

**管理方式**:
- **Toolkit**: 工具包管理器
- **工具组**: 支持分组激活/停用
- **MCP 集成**: 支持 Model Context Protocol

#### 2.2.4 Memory（记忆）

**类型**:
1. **短期记忆**: 当前对话历史（InMemoryMemory）
2. **长期记忆**: 持久化的知识（Mem0LongTermMemory）

#### 2.2.5 Pipeline（工作流）

**定义**: 组织多个 Agent 协作的方式。

**类型**:
- **SequentialPipeline**: 顺序执行
- **FanoutPipeline**: 并行执行
- **MsgHub**: 消息中心（发布-订阅）

---

## 3. 模块详细分析

### 3.1 Agent 模块

**文件位置**: `src/agentscope/agent/`

#### 3.1.1 文件结构

```
agent/
├── __init__.py                # 导出接口
├── _agent_base.py            # AgentBase 基类
├── _agent_meta.py            # Agent 元类
├── _react_agent_base.py      # ReActAgentBase 基类
├── _react_agent.py           # ReActAgent 实现
├── _user_agent.py            # UserAgent 实现
└── _user_input.py            # 用户输入处理
```

#### 3.1.2 AgentBase 类

**文件**: `_agent_base.py`

**继承关系**:
```python
AgentBase(StateModule, ABC)
```

**核心属性**:
```python
class AgentBase:
    name: str                        # Agent 名称
    memory: MemoryBase              # 记忆系统
    subscribers: dict               # 订阅者列表
    _hooks: dict                    # 钩子函数
```

**核心方法**:

1. **observe() - 观察消息**
```python
async def observe(self, msg: Msg | list[Msg] | None) -> None:
    """接收消息但不生成回复

    调用流程:
    1. 执行 pre_observe 钩子
    2. 将消息添加到 memory
    3. 执行 post_observe 钩子
    """
```

2. **reply() - 生成回复**
```python
async def reply(self, *args: Any, **kwargs: Any) -> Msg:
    """生成回复（抽象方法，子类必须实现）

    这是 Agent 的核心逻辑所在
    """
```

3. **__call__() - 调用 Agent**
```python
async def __call__(self, *args: Any, **kwargs: Any) -> Msg:
    """执行完整的 Agent 流程

    调用流程:
    1. 执行 pre_reply 钩子
    2. 调用 reply() 生成回复
    3. 执行 post_reply 钩子
    4. 广播消息到订阅者
    5. 打印消息
    """
```

4. **print() - 打印消息**
```python
async def print(self, msg: Msg, last: bool = True) -> None:
    """显示消息（支持流式输出）

    调用流程:
    1. 执行 pre_print 钩子
    2. 格式化并打印消息
    3. 执行 post_print 钩子
    """
```

**钩子系统**:

AgentBase 支持 6 种钩子:
```python
HOOK_TYPES = [
    "pre_reply",    # reply() 执行前
    "post_reply",   # reply() 执行后
    "pre_print",    # print() 执行前
    "post_print",   # print() 执行后
    "pre_observe",  # observe() 执行前
    "post_observe", # observe() 执行后
]
```

**注册钩子**:
```python
# 类级钩子（影响所有实例）
AgentBase.register_class_hook(
    hook_type="pre_reply",
    hook_name="my_hook",
    hook_func=my_hook_function
)

# 实例级钩子（仅影响当前实例）
agent.register_instance_hook(
    hook_type="post_reply",
    hook_name="my_hook",
    hook_func=my_hook_function
)
```

**状态管理**:
```python
def state_dict(self) -> dict:
    """序列化 Agent 状态"""
    return {
        "name": self.name,
        "memory": self.memory.state_dict(),
        # ... 其他状态
    }

def load_state_dict(self, state_dict: dict) -> None:
    """加载 Agent 状态"""
    self.name = state_dict["name"]
    self.memory.load_state_dict(state_dict["memory"])
    # ... 其他状态
```

#### 3.1.3 ReActAgentBase 类

**文件**: `_react_agent_base.py`

**扩展钩子**:
```python
HOOK_TYPES = [
    *AgentBase.HOOK_TYPES,
    "pre_reasoning",   # 推理前
    "post_reasoning",  # 推理后
    "pre_acting",      # 执行前
    "post_acting",     # 执行后
]
```

**抽象方法**:
```python
async def _reasoning(self, *args, **kwargs) -> Any:
    """推理过程（由子类实现）"""

async def _acting(self, *args, **kwargs) -> Any:
    """执行过程（由子类实现）"""
```

#### 3.1.4 ReActAgent 类

**文件**: `_react_agent.py`

这是 AgentScope 最核心的 Agent 实现，支持完整的 ReAct 循环。

**初始化参数**:
```python
def __init__(
    self,
    name: str,                              # Agent 名称
    sys_prompt: str | None = None,          # 系统提示
    model: ChatModelBase,                   # LLM 模型
    memory: MemoryBase,                     # 短期记忆
    formatter: FormatterBase,               # 提示词格式化器
    toolkit: Toolkit | None = None,         # 工具包
    long_term_memory: LongTermMemoryBase | None = None,  # 长期记忆
    long_term_memory_mode: Literal["agent_control", "static_control", "both"] = "agent_control",
    knowledge: list[KnowledgeBase] | KnowledgeBase | None = None,  # 知识库
    enable_rewrite_query: bool = False,     # 启用查询重写
    plan_notebook: PlanNotebook | None = None,  # 计划笔记本
    max_iters: int = 10,                    # 最大迭代次数
):
```

**核心执行流程**:

```python
async def reply(
    self,
    msg: Msg | list[Msg] | None = None,
    structured_model: type[BaseModel] | None = None
) -> Msg:
    """ReAct 主循环

    执行流程:
    1. 将输入消息添加到记忆
    2. 从长期记忆检索相关信息
    3. 从知识库检索相关信息
    4. 进入 ReAct 循环（最多 max_iters 次）:
       a. 推理：调用 LLM 生成回复/工具调用
       b. 执行：执行工具调用
       c. 如果有最终回复，退出循环
    5. 记录到长期记忆
    6. 返回最终回复
    """

    # 1. 记录输入
    await self.memory.add(msg)

    # 2. 长期记忆检索
    if self._static_control and self.long_term_memory:
        retrieved = await self.long_term_memory.retrieve(
            query=msg_text,
            agent_id=self.agent_id
        )
        if retrieved:
            await self.memory.add(Msg("system", retrieved, "system"))

    # 3. 知识库检索
    if self.knowledge:
        for kb in self.knowledge:
            docs = await kb.retrieve(query=msg_text)
            # 添加到记忆

    # 4. ReAct 循环
    for iteration in range(self.max_iters):
        # 4a. 推理
        msg_reasoning = await self._reasoning(structured_model)

        # 4b. 提取工具调用
        tool_calls = msg_reasoning.get_content_blocks("tool_use")

        if tool_calls:
            # 4c. 执行工具
            acting_responses = await self._acting(tool_calls)
            await self.memory.add(acting_responses)
        else:
            # 没有工具调用，说明是最终回复
            reply_msg = msg_reasoning
            break

    # 5. 记录到长期记忆
    if self._static_control and self.long_term_memory:
        await self.long_term_memory.record(
            messages=[msg, reply_msg],
            agent_id=self.agent_id
        )

    return reply_msg
```

**推理过程** (`_reasoning`):
```python
async def _reasoning(
    self,
    structured_model: type[BaseModel] | None = None
) -> Msg:
    """调用 LLM 进行推理

    执行流程:
    1. 获取计划提示（如果有）
    2. 格式化消息历史
    3. 调用 LLM
    4. 处理流式/非流式响应
    5. 返回响应消息
    """
    # 执行 pre_reasoning 钩子

    # 获取记忆
    memory = await self.memory.get_memory()

    # 添加系统提示
    if self.sys_prompt:
        memory = [Msg("system", self.sys_prompt, "system")] + memory

    # 添加计划提示
    if self.plan_notebook:
        plan_hint = await self.plan_notebook.get_current_hint()
        if plan_hint:
            memory.append(plan_hint)

    # 格式化消息
    formatted = await self.formatter.format(
        msgs=memory,
        tools=self.toolkit.json_schemas if self.toolkit else None
    )

    # 调用模型
    response = await self.model(
        messages=formatted,
        structured_model=structured_model
    )

    # 处理响应（流式或非流式）
    msg = await self._process_model_response(response)

    # 执行 post_reasoning 钩子

    return msg
```

**执行过程** (`_acting`):
```python
async def _acting(
    self,
    tool_calls: list[ToolUseBlock]
) -> list[Msg]:
    """执行工具调用

    执行流程:
    1. 并行执行所有工具调用
    2. 收集结果
    3. 返回工具结果消息
    """
    # 执行 pre_acting 钩子

    # 并行执行工具
    tasks = [
        self.toolkit.call_tool_function(tool_call)
        for tool_call in tool_calls
    ]

    results = await asyncio.gather(*tasks)

    # 构造工具结果消息
    tool_result_msgs = [
        Msg(
            name=self.name,
            content=[ToolResultBlock(
                type="tool_result",
                id=tool_call["id"],
                output=result.content
            )],
            role="assistant"
        )
        for tool_call, result in zip(tool_calls, results)
    ]

    # 执行 post_acting 钩子

    return tool_result_msgs
```

**长期记忆模式**:

1. **agent_control**: Agent 自主决定何时检索和记录
   - Agent 可以调用 `retrieve_from_long_term_memory` 和 `record_to_long_term_memory` 工具

2. **static_control**: 每次自动检索和记录
   - 每次 `reply()` 开始时自动检索
   - 每次 `reply()` 结束时自动记录

3. **both**: 两者结合
   - 自动检索和记录
   - Agent 也可以主动调用工具

**知识库集成**:

```python
# 初始化时提供知识库
agent = ReActAgent(
    knowledge=[kb1, kb2, kb3],  # 多个知识库
    enable_rewrite_query=True   # 启用查询重写
)

# 检索时会:
# 1. 合并所有知识库的检索结果
# 2. 如果启用查询重写，先让 LLM 重写查询
```

**计划集成**:

```python
# 初始化时提供计划笔记本
plan_notebook = PlanNotebook()
agent = ReActAgent(
    plan_notebook=plan_notebook
)

# Agent 会自动:
# 1. 注册计划相关的工具函数
# 2. 在推理时添加计划提示
# 3. 根据计划状态调整行为
```

#### 3.1.5 UserAgent 类

**文件**: `_user_agent.py`

**功能**: 与用户进行交互的 Agent。

**输入源**:
```python
class UserAgent(AgentBase):
    # 类级输入方法
    _class_input_method: UserInputMethod | None = None

    # 实例级输入方法
    _instance_input_method: UserInputMethod | None = None
```

**使用示例**:
```python
# 默认从终端获取输入
user = UserAgent(name="user")

# 使用 Studio 输入
from agentscope.agent import StudioUserInput
UserAgent.override_class_input_method(
    StudioUserInput(studio_url="http://localhost:5000")
)

# 自定义输入方法
def custom_input(prompt: str) -> str:
    return input(f"[Custom] {prompt}: ")

user.override_instance_input_method(custom_input)
```

---

### 3.2 Model 模块

**文件位置**: `src/agentscope/model/`

#### 3.2.1 文件结构

```
model/
├── __init__.py                  # 导出接口
├── _model_base.py              # 模型基类
├── _model_response.py          # 响应类
├── _model_usage.py             # 使用统计
├── _dashscope_model.py         # 阿里云通义千问
├── _openai_model.py            # OpenAI
├── _anthropic_model.py         # Anthropic Claude
├── _ollama_model.py            # Ollama
├── _gemini_model.py            # Google Gemini
└── ...                         # 其他模型
```

#### 3.2.2 ChatModelBase 类

**文件**: `_model_base.py`

**抽象基类**:
```python
class ChatModelBase(StateModule, ABC):
    model_name: str              # 模型名称
    stream: bool = False         # 是否流式输出
```

**核心方法**:
```python
@abstractmethod
async def __call__(
    self,
    messages: list[dict],
    tools: list[dict] | None = None,
    tool_choice: str | None = None,
    structured_model: type[BaseModel] | None = None,
    **kwargs
) -> ChatResponse | AsyncGenerator[ChatResponse, None]:
    """调用模型生成响应

    参数:
        messages: 消息历史
        tools: 可用工具列表（JSON Schema）
        tool_choice: 工具选择策略
        structured_model: 结构化输出模型（Pydantic）

    返回:
        ChatResponse 或流式生成器
    """
```

**工具选择策略**:
```python
# tool_choice 支持的值:
"auto"      # 模型自动决定
"none"      # 不调用工具
"any"       # 必须调用某个工具
"required"  # 必须调用工具
"function_name"  # 调用指定工具
```

#### 3.2.3 ChatResponse 类

**文件**: `_model_response.py`

**数据结构**:
```python
@dataclass
class ChatResponse:
    content: Sequence[ContentBlock]  # 内容块列表
    id: str                          # 响应 ID
    created_at: str                  # 创建时间
    type: Literal["chat"]            # 类型
    usage: ChatUsage | None          # 使用统计
    metadata: dict | None            # 元数据
    stream: bool = False             # 是否流式
    is_last: bool = True             # 是否最后一个
```

**内容块类型**:
- `TextBlock`: 文本内容
- `ThinkingBlock`: 思考过程（Claude 等模型）
- `ToolUseBlock`: 工具调用
- `AudioBlock`: 音频内容

**转换为 Msg**:
```python
def to_msg(self, name: str, role: str = "assistant") -> Msg:
    """将响应转换为 Msg 对象"""
    return Msg(
        name=name,
        content=self.content,
        role=role,
        metadata=self.metadata
    )
```

#### 3.2.4 具体模型实现

**1. DashScopeChatModel (阿里云通义千问)**

```python
class DashScopeChatModel(ChatModelBase):
    def __init__(
        self,
        model_name: str,                    # qwen-max, qwen-plus 等
        api_key: str,
        stream: bool = False,
        **kwargs
    ):
        """初始化通义千问模型"""

    async def __call__(
        self,
        messages: list[dict],
        **kwargs
    ) -> ChatResponse | AsyncGenerator:
        """调用 DashScope API"""
```

**特性**:
- 支持多模态（文本、图片、音频、视频）
- 支持 Qwen-Omni 音频模型
- 支持并行工具调用

**2. OpenAIChatModel**

```python
class OpenAIChatModel(ChatModelBase):
    def __init__(
        self,
        model_name: str,                    # gpt-4, gpt-3.5-turbo 等
        api_key: str,
        base_url: str | None = None,
        stream: bool = False,
        **kwargs
    ):
        """初始化 OpenAI 模型"""
```

**特性**:
- 支持 GPT 系列模型
- 支持 GPT-4o Audio 模型
- 支持结构化输出（structured_model）
- 支持并行工具调用

**3. AnthropicChatModel**

```python
class AnthropicChatModel(ChatModelBase):
    def __init__(
        self,
        model_name: str,                    # claude-3-opus 等
        api_key: str,
        stream: bool = False,
        **kwargs
    ):
        """初始化 Anthropic Claude 模型"""
```

**特性**:
- 支持 Claude 系列模型
- 支持思考块（ThinkingBlock）
- 支持并行工具调用

**4. OllamaChatModel**

```python
class OllamaChatModel(ChatModelBase):
    def __init__(
        self,
        model_name: str,                    # llama2, mistral 等
        base_url: str = "http://localhost:11434",
        stream: bool = False,
        **kwargs
    ):
        """初始化 Ollama 本地模型"""
```

**特性**:
- 支持本地部署的开源模型
- 支持工具调用（部分模型）

**5. GeminiChatModel**

```python
class GeminiChatModel(ChatModelBase):
    def __init__(
        self,
        model_name: str,                    # gemini-pro 等
        api_key: str,
        stream: bool = False,
        **kwargs
    ):
        """初始化 Google Gemini 模型"""
```

**特性**:
- 支持 Gemini 系列模型
- 支持多模态
- 支持工具调用

#### 3.2.5 流式处理

**流式响应示例**:
```python
# 调用流式模型
model = DashScopeChatModel(model_name="qwen-max", stream=True)

async for response in model(messages):
    # response.stream == True
    # response.is_last 指示是否为最后一个块

    # 累积式内容
    # 每个响应包含从头到当前的所有内容
    print(response.content)

    if response.is_last:
        # 完整响应
        final_response = response
```

---

### 3.3 Message 模块

**文件位置**: `src/agentscope/message/`

#### 3.3.1 文件结构

```
message/
├── __init__.py              # 导出接口
├── _message_base.py        # Msg 类
└── _message_block.py       # 内容块定义
```

#### 3.3.2 Msg 类

**文件**: `_message_base.py`

**核心设计**:
```python
class Msg:
    """消息类 - Agent 间通信的基本单位"""

    def __init__(
        self,
        name: str,                                    # 发送者名称
        content: str | Sequence[ContentBlock],        # 内容
        role: Literal["user", "assistant", "system"], # 角色
        metadata: dict | None = None,                 # 元数据
        timestamp: str | None = None,                 # 时间戳
        invocation_id: str | None = None              # 调用ID
    ):
        self.name = name
        self.role = role
        self.metadata = metadata or {}
        self.timestamp = timestamp or datetime.now().isoformat()
        self.invocation_id = invocation_id or str(uuid.uuid4())
        self.id = str(uuid.uuid4())  # 消息唯一ID

        # 处理内容
        if isinstance(content, str):
            self.content = [TextBlock(type="text", text=content)]
        else:
            self.content = list(content)
```

**关键方法**:

1. **get_text_content() - 提取文本内容**
```python
def get_text_content(self) -> str | None:
    """提取所有文本块的内容

    返回:
        合并后的文本，如果没有文本块则返回 None
    """
    text_blocks = [
        block["text"]
        for block in self.content
        if block["type"] == "text"
    ]
    return "\n".join(text_blocks) if text_blocks else None
```

2. **get_content_blocks() - 获取指定类型的块**
```python
def get_content_blocks(
    self,
    block_type: str | None = None
) -> list[ContentBlock]:
    """获取指定类型的内容块

    参数:
        block_type: 块类型，如 "text", "tool_use" 等
                   为 None 时返回所有块

    返回:
        内容块列表
    """
    if block_type is None:
        return self.content

    return [
        block
        for block in self.content
        if block["type"] == block_type
    ]
```

3. **has_content_blocks() - 检查是否包含指定块**
```python
def has_content_blocks(
    self,
    block_type: str | None = None
) -> bool:
    """检查是否包含指定类型的内容块"""
    return len(self.get_content_blocks(block_type)) > 0
```

4. **to_dict() - 序列化**
```python
def to_dict(self) -> dict:
    """序列化为字典（用于保存和传输）"""
    return {
        "id": self.id,
        "name": self.name,
        "content": self.content,  # 已经是可序列化的
        "role": self.role,
        "metadata": self.metadata,
        "timestamp": self.timestamp,
        "invocation_id": self.invocation_id,
    }
```

5. **from_dict() - 反序列化**
```python
@classmethod
def from_dict(cls, json_data: dict) -> "Msg":
    """从字典反序列化"""
    msg = cls(
        name=json_data["name"],
        content=json_data["content"],
        role=json_data["role"],
        metadata=json_data.get("metadata"),
        timestamp=json_data.get("timestamp"),
        invocation_id=json_data.get("invocation_id"),
    )
    msg.id = json_data["id"]
    return msg
```

#### 3.3.3 内容块类型

**文件**: `_message_block.py`

所有内容块都使用 TypedDict 定义，确保类型安全。

**1. TextBlock - 文本块**
```python
class TextBlock(TypedDict):
    type: Literal["text"]
    text: str
```

**使用示例**:
```python
msg = Msg(
    name="assistant",
    content=[TextBlock(type="text", text="Hello!")],
    role="assistant"
)
```

**2. ThinkingBlock - 思考块**
```python
class ThinkingBlock(TypedDict):
    type: Literal["thinking"]
    thinking: str
```

**用途**: Claude 等模型的推理过程

**3. ToolUseBlock - 工具调用块**
```python
class ToolUseBlock(TypedDict):
    type: Literal["tool_use"]
    id: str                      # 工具调用 ID
    name: str                    # 工具名称
    input: dict[str, object]     # 工具参数
```

**使用示例**:
```python
tool_call = ToolUseBlock(
    type="tool_use",
    id="call_123",
    name="execute_python_code",
    input={"code": "print('hello')"}
)
```

**4. ToolResultBlock - 工具结果块**
```python
class ToolResultBlock(TypedDict):
    type: Literal["tool_result"]
    id: str                      # 对应的工具调用 ID
    output: str | List[TextBlock | ImageBlock | AudioBlock]
```

**使用示例**:
```python
tool_result = ToolResultBlock(
    type="tool_result",
    id="call_123",
    output="hello\n"  # 或者多模态输出
)
```

**5. ImageBlock - 图片块**
```python
class Base64Source(TypedDict):
    type: Literal["base64"]
    media_type: str              # "image/png", "image/jpeg" 等
    data: str                    # Base64 编码的数据

class URLSource(TypedDict):
    type: Literal["url"]
    url: str                     # 图片 URL

class ImageBlock(TypedDict):
    type: Literal["image"]
    source: Base64Source | URLSource
```

**使用示例**:
```python
# Base64 图片
img = ImageBlock(
    type="image",
    source=Base64Source(
        type="base64",
        media_type="image/png",
        data="iVBORw0KGgo..."
    )
)

# URL 图片
img = ImageBlock(
    type="image",
    source=URLSource(
        type="url",
        url="https://example.com/image.png"
    )
)
```

**6. AudioBlock - 音频块**
```python
class AudioBlock(TypedDict):
    type: Literal["audio"]
    source: Base64Source | URLSource
```

**7. VideoBlock - 视频块**
```python
class VideoBlock(TypedDict):
    type: Literal["video"]
    source: Base64Source | URLSource
```

#### 3.3.4 消息使用模式

**1. 简单文本消息**
```python
msg = Msg("user", "Hello, how are you?", "user")
```

**2. 多模态消息**
```python
msg = Msg(
    name="user",
    content=[
        TextBlock(type="text", text="What's in this image?"),
        ImageBlock(
            type="image",
            source=URLSource(type="url", url="https://example.com/cat.jpg")
        )
    ],
    role="user"
)
```

**3. 工具调用消息**
```python
msg = Msg(
    name="assistant",
    content=[
        ThinkingBlock(
            type="thinking",
            thinking="I need to execute Python code to solve this"
        ),
        ToolUseBlock(
            type="tool_use",
            id="call_1",
            name="execute_python_code",
            input={"code": "2 + 2"}
        )
    ],
    role="assistant"
)
```

**4. 工具结果消息**
```python
msg = Msg(
    name="assistant",
    content=[
        ToolResultBlock(
            type="tool_result",
            id="call_1",
            output="4"
        )
    ],
    role="assistant"
)
```

---

### 3.4 Tool 模块

**文件位置**: `src/agentscope/tool/`

#### 3.4.1 文件结构

```
tool/
├── __init__.py                          # 导出接口
├── _toolkit.py                          # Toolkit 工具包管理器
├── _response.py                         # ToolResponse 响应类
├── _registered_tool_function.py         # 已注册工具函数
├── _async_wrapper.py                    # 异步包装器
├── _coding/                             # 代码执行工具
│   ├── _python.py                       # Python 代码执行
│   └── _shell.py                        # Shell 命令执行
├── _text_file/                          # 文件操作工具
│   ├── _view_text_file.py               # 查看文件
│   ├── _write_text_file.py              # 写入文件
│   ├── _insert_text_file.py             # 插入文件内容
│   └── _utils.py
└── _multi_modality/                     # 多模态工具
    ├── _dashscope_tools.py              # 通义多模态工具
    └── _openai_tools.py                 # OpenAI 多模态工具
```

#### 3.4.2 Toolkit 类

**文件**: `_toolkit.py`

Toolkit 是 AgentScope 的核心工具管理器，负责注册、管理和执行工具函数。

**核心设计**:
```python
class Toolkit(StateModule):
    """工具包管理器

    特性:
    - 函数级管理：注册/移除单个工具函数
    - 组级管理：将工具分组，支持整组激活/停用
    - MCP 集成：支持 Model Context Protocol
    - 统一接口：所有工具都返回 ToolResponse
    - 流式支持：支持流式工具函数
    """

    def __init__(self):
        super().__init__()
        self.tool_functions: dict[str, RegisteredToolFunction] = {}
        self.tool_groups: dict[str, ToolGroup] = {
            "basic": ToolGroup(
                name="basic",
                description="Basic tools",
                active=True  # basic 组始终激活
            )
        }
```

**函数级管理**:

1. **注册工具函数**
```python
def register_tool_function(
    self,
    tool_func: ToolFunction,              # 工具函数
    group_name: str = "basic",            # 所属组
    preset_kwargs: dict | None = None,    # 预设参数
    func_description: str | None = None,  # 自定义描述
    json_schema: dict | None = None,      # 自定义 JSON Schema
    postprocess_func: Callable | None = None  # 后处理函数
) -> None:
    """注册工具函数

    自动功能:
    - 从 docstring 提取描述和参数说明
    - 生成 JSON Schema
    - 包装异步/同步函数
    - 设置预设参数
    """
    # 解析函数签名和文档
    parsed = parse_docstring(tool_func)

    # 生成 JSON Schema
    if json_schema is None:
        json_schema = generate_json_schema(tool_func, parsed)

    # 包装函数（确保异步）
    wrapped_func = ensure_async(tool_func)

    # 注册
    self.tool_functions[tool_func.__name__] = RegisteredToolFunction(
        func=wrapped_func,
        group_name=group_name,
        json_schema=json_schema,
        preset_kwargs=preset_kwargs,
        postprocess_func=postprocess_func
    )
```

**使用示例**:
```python
toolkit = Toolkit()

# 注册简单函数
def add(a: int, b: int) -> int:
    """Add two numbers

    Args:
        a: First number
        b: Second number
    """
    return a + b

toolkit.register_tool_function(add)

# 注册异步函数
async def async_search(query: str) -> str:
    """Search the web

    Args:
        query: Search query
    """
    # 异步搜索逻辑
    return "search results"

toolkit.register_tool_function(async_search, group_name="web")

# 注册带预设参数的函数
toolkit.register_tool_function(
    execute_shell_command,
    preset_kwargs={"timeout": 30}
)

# 注册带后处理的函数
def postprocess(result: ToolResponse) -> ToolResponse:
    # 处理结果
    return result

toolkit.register_tool_function(
    my_tool,
    postprocess_func=postprocess
)
```

2. **移除工具函数**
```python
def remove_tool_function(self, tool_name: str) -> None:
    """移除已注册的工具函数"""
    if tool_name in self.tool_functions:
        del self.tool_functions[tool_name]
```

**组级管理**:

1. **创建工具组**
```python
def create_tool_group(
    self,
    group_name: str,
    description: str,
    active: bool = False,
    notes: str | None = None
) -> None:
    """创建工具组

    工具组用于:
    - 组织相关工具
    - 批量激活/停用
    - Agent 动态选择
    """
    self.tool_groups[group_name] = ToolGroup(
        name=group_name,
        description=description,
        active=active,
        notes=notes
    )
```

2. **更新工具组状态**
```python
def update_tool_groups(
    self,
    group_names: list[str],
    active: bool
) -> None:
    """批量激活/停用工具组"""
    for group_name in group_names:
        if group_name in self.tool_groups:
            self.tool_groups[group_name].active = active
```

**使用示例**:
```python
toolkit = Toolkit()

# 创建工具组
toolkit.create_tool_group(
    group_name="web",
    description="Web search and browsing tools",
    active=False
)

toolkit.create_tool_group(
    group_name="file",
    description="File operations",
    active=True
)

# 注册工具到组
toolkit.register_tool_function(web_search, group_name="web")
toolkit.register_tool_function(read_file, group_name="file")

# 激活工具组
toolkit.update_tool_groups(["web"], active=True)

# 停用工具组
toolkit.update_tool_groups(["file"], active=False)
```

**MCP 集成**:

```python
async def register_mcp_client(
    self,
    mcp_client: MCPClientBase,
    group_name: str = "basic",
    enable_funcs: list[str] | None = None,
    disable_funcs: list[str] | None = None
) -> None:
    """注册 MCP 服务器的工具函数

    参数:
        mcp_client: MCP 客户端
        group_name: 工具组名称
        enable_funcs: 启用的函数列表（None 表示全部）
        disable_funcs: 禁用的函数列表
    """
    # 获取 MCP 服务器的工具列表
    tools = await mcp_client.list_tools()

    # 过滤工具
    for tool in tools:
        if enable_funcs and tool.name not in enable_funcs:
            continue
        if disable_funcs and tool.name in disable_funcs:
            continue

        # 创建 MCP 工具函数
        mcp_func = await mcp_client.get_callable_function(tool.name)

        # 注册到 Toolkit
        self.register_tool_function(
            mcp_func,
            group_name=group_name
        )
```

**使用示例**:
```python
from agentscope.mcp import HttpStatelessClient

# 创建 MCP 客户端
mcp_client = HttpStatelessClient(
    name="gaode_mcp",
    url="https://mcp.amap.com/mcp?key=YOUR_KEY"
)

# 注册 MCP 工具
await toolkit.register_mcp_client(
    mcp_client,
    group_name="map",
    enable_funcs=["maps_geo", "maps_search"]  # 只启用这两个
)
```

**工具调用**:

```python
async def call_tool_function(
    self,
    tool_call: ToolUseBlock
) -> AsyncGenerator[ToolResponse, None]:
    """执行工具函数（统一流式接口）

    执行流程:
    1. 查找工具函数
    2. 合并预设参数
    3. 执行函数
    4. 应用后处理
    5. 返回结果（流式或非流式）

    参数:
        tool_call: 工具调用块

    返回:
        流式生成器（即使非流式工具也包装为生成器）
    """
    tool_name = tool_call["name"]

    # 1. 查找工具
    if tool_name not in self.tool_functions:
        yield ToolResponse(
            content=[TextBlock(
                type="text",
                text=f"Error: Tool '{tool_name}' not found"
            )],
            stream=False,
            is_last=True
        )
        return

    registered = self.tool_functions[tool_name]

    # 2. 合并参数
    kwargs = {**registered.preset_kwargs, **tool_call["input"]}

    # 3. 执行函数
    try:
        result = await registered.func(**kwargs)

        # 4. 判断是否流式
        if isinstance(result, AsyncGenerator):
            # 流式工具
            async for response in result:
                # 应用后处理
                if registered.postprocess_func:
                    response = registered.postprocess_func(response)
                yield response
        else:
            # 非流式工具
            if not isinstance(result, ToolResponse):
                result = ToolResponse(
                    content=[TextBlock(type="text", text=str(result))],
                    stream=False,
                    is_last=True
                )

            # 应用后处理
            if registered.postprocess_func:
                result = registered.postprocess_func(result)

            yield result

    except Exception as e:
        yield ToolResponse(
            content=[TextBlock(
                type="text",
                text=f"Error executing tool: {str(e)}"
            )],
            stream=False,
            is_last=True
        )
```

**元工具函数**:

Toolkit 提供元工具函数，允许 Agent 动态选择装备的工具组。

```python
def reset_equipped_tools(
    self,
    group_names: list[str]
) -> ToolResponse:
    """重置装备的工具组（元工具）

    Agent 可以调用此函数来:
    - 选择需要的工具组
    - 停用不需要的工具组
    """
    # 停用所有非 basic 组
    for group in self.tool_groups.values():
        if group.name != "basic":
            group.active = False

    # 激活指定的组
    self.update_tool_groups(group_names, active=True)

    return ToolResponse(
        content=[TextBlock(
            type="text",
            text=f"Equipped tools: {', '.join(group_names)}"
        )]
    )

# 自动注册为工具函数
toolkit.register_tool_function(
    toolkit.reset_equipped_tools,
    group_name="basic"
)
```

**JSON Schema 生成**:

```python
@property
def json_schemas(self) -> list[dict]:
    """生成当前激活工具的 JSON Schema 列表

    返回的 Schema 可以直接传递给 LLM
    """
    schemas = []

    for tool_name, registered in self.tool_functions.items():
        # 只包含激活组的工具
        if self.tool_groups[registered.group_name].active:
            schemas.append(registered.json_schema)

    return schemas
```

**使用示例**:
```python
# 获取 JSON Schemas
schemas = toolkit.json_schemas

# 传递给模型
response = await model(
    messages=messages,
    tools=schemas,
    tool_choice="auto"
)
```

#### 3.4.3 ToolResponse 类

**文件**: `_response.py`

```python
@dataclass
class ToolResponse:
    """工具函数响应

    统一的工具返回格式，支持:
    - 文本、图片、音频等多模态内容
    - 流式输出
    - 元数据传递
    """
    content: Sequence[ContentBlock]    # 内容块
    metadata: dict | None = None       # 元数据（Agent 内部使用）
    stream: bool = False               # 是否流式
    is_last: bool = True               # 是否最后一个块
```

**使用示例**:

1. **简单文本响应**
```python
def my_tool() -> ToolResponse:
    return ToolResponse(
        content=[TextBlock(type="text", text="result")]
    )
```

2. **多模态响应**
```python
def generate_image(prompt: str) -> ToolResponse:
    # 生成图片
    image_data = generate(prompt)

    return ToolResponse(
        content=[
            TextBlock(type="text", text="Image generated successfully"),
            ImageBlock(
                type="image",
                source=Base64Source(
                    type="base64",
                    media_type="image/png",
                    data=image_data
                )
            )
        ]
    )
```

3. **流式响应**
```python
async def streaming_tool(query: str) -> AsyncGenerator[ToolResponse, None]:
    """流式工具函数"""
    chunks = process_streaming(query)

    for i, chunk in enumerate(chunks):
        is_last = (i == len(chunks) - 1)

        yield ToolResponse(
            content=[TextBlock(type="text", text=chunk)],
            stream=True,
            is_last=is_last
        )
```

4. **带元数据的响应**
```python
def search(query: str) -> ToolResponse:
    results = perform_search(query)

    return ToolResponse(
        content=[TextBlock(type="text", text=results["summary"])],
        metadata={
            "sources": results["sources"],
            "confidence": results["confidence"]
        }
    )
```

#### 3.4.4 内置工具函数

**1. 代码执行工具**

**execute_python_code**:
```python
def execute_python_code(
    code: str,
    timeout: int = 30
) -> ToolResponse:
    """Execute Python code in a sandboxed environment

    Args:
        code: Python code to execute
        timeout: Execution timeout in seconds

    Returns:
        Execution output
    """
    # 在隔离环境中执行代码
    result = subprocess.run(
        ["python", "-c", code],
        capture_output=True,
        timeout=timeout,
        text=True
    )

    output = result.stdout or result.stderr

    return ToolResponse(
        content=[TextBlock(type="text", text=output)]
    )
```

**execute_shell_command**:
```python
def execute_shell_command(
    command: str,
    timeout: int = 30
) -> ToolResponse:
    """Execute shell command

    Args:
        command: Shell command to execute
        timeout: Execution timeout in seconds
    """
    result = subprocess.run(
        command,
        shell=True,
        capture_output=True,
        timeout=timeout,
        text=True
    )

    output = result.stdout or result.stderr

    return ToolResponse(
        content=[TextBlock(type="text", text=output)]
    )
```

**2. 文件操作工具**

**view_text_file**:
```python
def view_text_file(
    file_path: str,
    start_line: int | None = None,
    end_line: int | None = None
) -> ToolResponse:
    """View content of a text file

    Args:
        file_path: Path to the file
        start_line: Starting line number (1-indexed)
        end_line: Ending line number (inclusive)
    """
    with open(file_path, "r", encoding="utf-8") as f:
        lines = f.readlines()

    if start_line or end_line:
        start = (start_line - 1) if start_line else 0
        end = end_line if end_line else len(lines)
        lines = lines[start:end]

    content = "".join(lines)

    return ToolResponse(
        content=[TextBlock(type="text", text=content)]
    )
```

**write_text_file**:
```python
def write_text_file(
    file_path: str,
    content: str
) -> ToolResponse:
    """Write content to a text file

    Args:
        file_path: Path to the file
        content: Content to write
    """
    with open(file_path, "w", encoding="utf-8") as f:
        f.write(content)

    return ToolResponse(
        content=[TextBlock(
            type="text",
            text=f"Successfully wrote to {file_path}"
        )]
    )
```

**insert_text_file**:
```python
def insert_text_file(
    file_path: str,
    line_number: int,
    content: str
) -> ToolResponse:
    """Insert content at specific line in a text file

    Args:
        file_path: Path to the file
        line_number: Line number to insert at (1-indexed)
        content: Content to insert
    """
    with open(file_path, "r", encoding="utf-8") as f:
        lines = f.readlines()

    lines.insert(line_number - 1, content + "\n")

    with open(file_path, "w", encoding="utf-8") as f:
        f.writelines(lines)

    return ToolResponse(
        content=[TextBlock(
            type="text",
            text=f"Successfully inserted at line {line_number}"
        )]
    )
```

**3. 多模态工具**

**DashScope 工具**:

```python
def dashscope_text_to_image(
    prompt: str,
    model: str = "wanx-v1",
    size: str = "1024*1024"
) -> ToolResponse:
    """Generate image from text using DashScope

    Args:
        prompt: Text description
        model: Model name
        size: Image size
    """
    # 调用通义万相 API
    response = dashscope.ImageSynthesis.call(
        model=model,
        prompt=prompt,
        n=1,
        size=size
    )

    image_url = response.output.results[0].url

    return ToolResponse(
        content=[
            TextBlock(type="text", text="Image generated"),
            ImageBlock(
                type="image",
                source=URLSource(type="url", url=image_url)
            )
        ]
    )
```

```python
def dashscope_text_to_audio(
    text: str,
    model: str = "cosyvoice-v1",
    voice: str = "longxiaochun"
) -> ToolResponse:
    """Convert text to speech using DashScope

    Args:
        text: Text to convert
        model: TTS model
        voice: Voice type
    """
    # 调用通义听悟 API
    response = dashscope.AudioSynthesis.call(
        model=model,
        text=text,
        voice=voice
    )

    audio_data = response.output.audio

    return ToolResponse(
        content=[
            TextBlock(type="text", text="Audio generated"),
            AudioBlock(
                type="audio",
                source=Base64Source(
                    type="base64",
                    media_type="audio/wav",
                    data=audio_data
                )
            )
        ]
    )
```

**OpenAI 工具**:

```python
def openai_text_to_image(
    prompt: str,
    model: str = "dall-e-3",
    size: str = "1024x1024"
) -> ToolResponse:
    """Generate image using DALL-E

    Args:
        prompt: Text description
        model: Model name (dall-e-2 or dall-e-3)
        size: Image size
    """
    response = openai.Image.create(
        model=model,
        prompt=prompt,
        size=size,
        n=1
    )

    image_url = response.data[0].url

    return ToolResponse(
        content=[
            TextBlock(type="text", text="Image generated"),
            ImageBlock(
                type="image",
                source=URLSource(type="url", url=image_url)
            )
        ]
    )
```

```python
def openai_audio_to_text(
    audio_file: str,
    model: str = "whisper-1"
) -> ToolResponse:
    """Transcribe audio using Whisper

    Args:
        audio_file: Path to audio file
        model: Whisper model
    """
    with open(audio_file, "rb") as f:
        response = openai.Audio.transcribe(
            model=model,
            file=f
        )

    text = response.text

    return ToolResponse(
        content=[TextBlock(type="text", text=text)]
    )
```

#### 3.4.5 工具函数最佳实践

**1. 清晰的文档字符串**

```python
def my_tool(arg1: str, arg2: int = 10) -> ToolResponse:
    """Simple one-line description

    More detailed description if needed.
    Explain what the tool does and when to use it.

    Args:
        arg1: Description of arg1 (required)
        arg2: Description of arg2 (optional, default=10)

    Returns:
        ToolResponse with the result
    """
    # 实现
```

**2. 类型注解**

```python
# 使用完整的类型注解
def search(
    query: str,
    limit: int = 10,
    filters: dict[str, str] | None = None
) -> ToolResponse:
    """Search with type hints"""
```

**3. 错误处理**

```python
def risky_tool(param: str) -> ToolResponse:
    """Tool with error handling"""
    try:
        result = perform_operation(param)
        return ToolResponse(
            content=[TextBlock(type="text", text=result)]
        )
    except Exception as e:
        return ToolResponse(
            content=[TextBlock(
                type="text",
                text=f"Error: {str(e)}"
            )],
            metadata={"error": True}
        )
```

**4. 流式工具**

```python
async def streaming_tool(
    query: str
) -> AsyncGenerator[ToolResponse, None]:
    """Streaming tool example"""

    # 初始化
    yield ToolResponse(
        content=[TextBlock(type="text", text="Starting...")],
        stream=True,
        is_last=False
    )

    # 处理过程
    for i, chunk in enumerate(process(query)):
        yield ToolResponse(
            content=[TextBlock(type="text", text=chunk)],
            stream=True,
            is_last=False
        )

    # 完成
    yield ToolResponse(
        content=[TextBlock(type="text", text="Done!")],
        stream=True,
        is_last=True
    )
```

---

### 3.5 Memory 模块

**文件位置**: `src/agentscope/memory/`

#### 3.5.1 文件结构

```
memory/
├── __init__.py                    # 导出接口
├── _memory_base.py               # MemoryBase 基类
├── _in_memory_memory.py          # InMemoryMemory 实现
├── _long_term_memory_base.py     # 长期记忆基类
├── _mem0_long_term_memory.py     # Mem0 长期记忆
└── _mem0_utils.py                # Mem0 工具函数
```

#### 3.5.2 MemoryBase 类

**文件**: `_memory_base.py`

```python
class MemoryBase(StateModule, ABC):
    """记忆基类

    定义所有记忆系统的统一接口
    """

    @abstractmethod
    async def add(self, *args, **kwargs) -> None:
        """添加项目到记忆"""

    @abstractmethod
    async def delete(self, *args, **kwargs) -> None:
        """从记忆删除项目"""

    @abstractmethod
    async def retrieve(self, *args, **kwargs) -> None:
        """从记忆检索项目"""

    @abstractmethod
    async def size(self) -> int:
        """获取记忆大小"""

    @abstractmethod
    async def clear(self) -> None:
        """清空记忆"""

    @abstractmethod
    async def get_memory(self, *args, **kwargs) -> list[Msg]:
        """获取记忆内容（用于传递给 LLM）"""
```

#### 3.5.3 InMemoryMemory 类

**文件**: `_in_memory_memory.py`

最常用的记忆实现，将消息存储在内存列表中。

```python
class InMemoryMemory(MemoryBase):
    """内存记忆

    特性:
    - 简单高效
    - 支持去重
    - 完整的状态序列化
    """

    def __init__(self):
        super().__init__()
        self.content: list[Msg] = []
        self.register_state("content")

    async def add(
        self,
        memories: Msg | list[Msg] | None,
        allow_duplicates: bool = False
    ) -> None:
        """添加消息到记忆

        Args:
            memories: 消息或消息列表
            allow_duplicates: 是否允许重复（根据 id 判断）
        """
        if memories is None:
            return

        if not isinstance(memories, list):
            memories = [memories]

        # 去重
        if not allow_duplicates:
            existing_ids = {msg.id for msg in self.content}
            memories = [
                msg for msg in memories
                if msg.id not in existing_ids
            ]

        self.content.extend(memories)

    async def delete(self, index: int) -> None:
        """删除指定索引的消息"""
        if 0 <= index < len(self.content):
            del self.content[index]

    async def retrieve(self) -> list[Msg]:
        """检索所有消息"""
        return self.content.copy()

    async def size(self) -> int:
        """获取记忆大小"""
        return len(self.content)

    async def clear(self) -> None:
        """清空记忆"""
        self.content.clear()

    async def get_memory(self) -> list[Msg]:
        """获取记忆内容（用于 LLM）"""
        return self.content.copy()
```

**使用示例**:
```python
memory = InMemoryMemory()

# 添加消息
await memory.add(Msg("user", "Hello", "user"))
await memory.add(Msg("assistant", "Hi!", "assistant"))

# 获取记忆
messages = await memory.get_memory()

# 获取大小
size = await memory.size()

# 清空
await memory.clear()

# 状态序列化
state = memory.state_dict()
# 保存 state...

# 状态加载
new_memory = InMemoryMemory()
new_memory.load_state_dict(state)
```

#### 3.5.4 LongTermMemoryBase 类

**文件**: `_long_term_memory_base.py`

```python
class LongTermMemoryBase(StateModule, ABC):
    """长期记忆基类

    与短期记忆的区别:
    - 持久化存储
    - 语义检索
    - 跨会话
    """

    @abstractmethod
    async def record(
        self,
        messages: list[Msg],
        agent_id: str,
        **kwargs
    ) -> None:
        """记录消息到长期记忆

        Args:
            messages: 消息列表
            agent_id: Agent ID
        """

    @abstractmethod
    async def retrieve(
        self,
        query: str,
        agent_id: str,
        limit: int = 5,
        **kwargs
    ) -> str:
        """从长期记忆检索相关信息

        Args:
            query: 查询文本
            agent_id: Agent ID
            limit: 返回数量

        Returns:
            检索到的文本
        """

    @abstractmethod
    async def delete(
        self,
        agent_id: str,
        memory_id: str | None = None,
        **kwargs
    ) -> None:
        """删除长期记忆"""
```

#### 3.5.5 Mem0LongTermMemory 类

**文件**: `_mem0_long_term_memory.py`

集成 Mem0 平台的长期记忆实现。

```python
class Mem0LongTermMemory(LongTermMemoryBase):
    """Mem0 长期记忆

    特性:
    - 集成 Mem0 平台
    - 自动向量化
    - 语义检索
    - 多 Agent 支持
    """

    def __init__(
        self,
        api_key: str,
        org_id: str | None = None,
        project_id: str | None = None,
        **kwargs
    ):
        """初始化 Mem0 长期记忆

        Args:
            api_key: Mem0 API key
            org_id: Organization ID
            project_id: Project ID
        """
        from mem0 import MemoryClient

        self.client = MemoryClient(
            api_key=api_key,
            org_id=org_id,
            project_id=project_id
        )

    async def record(
        self,
        messages: list[Msg],
        agent_id: str,
        **kwargs
    ) -> None:
        """记录到 Mem0"""
        # 合并消息文本
        text = "\n".join([
            f"{msg.name}: {msg.get_text_content()}"
            for msg in messages
        ])

        # 记录到 Mem0
        self.client.add(
            text,
            user_id=agent_id,
            metadata={"agent_id": agent_id}
        )

    async def retrieve(
        self,
        query: str,
        agent_id: str,
        limit: int = 5,
        **kwargs
    ) -> str:
        """从 Mem0 检索"""
        # 搜索相关记忆
        results = self.client.search(
            query,
            user_id=agent_id,
            limit=limit
        )

        # 格式化结果
        retrieved_text = "\n\n".join([
            f"Memory {i+1}: {r['memory']}"
            for i, r in enumerate(results)
        ])

        return retrieved_text

    async def delete(
        self,
        agent_id: str,
        memory_id: str | None = None,
        **kwargs
    ) -> None:
        """删除 Mem0 记忆"""
        if memory_id:
            self.client.delete(memory_id)
        else:
            # 删除 Agent 的所有记忆
            self.client.delete_all(user_id=agent_id)
```

**使用示例**:
```python
# 创建长期记忆
ltm = Mem0LongTermMemory(
    api_key="your_mem0_api_key",
    org_id="your_org_id",
    project_id="your_project_id"
)

# 记录消息
await ltm.record(
    messages=[
        Msg("user", "My favorite color is blue", "user"),
        Msg("assistant", "Got it!", "assistant")
    ],
    agent_id="agent_001"
)

# 检索相关信息
retrieved = await ltm.retrieve(
    query="What is my favorite color?",
    agent_id="agent_001",
    limit=3
)
# 输出: "Memory 1: My favorite color is blue"

# 在 ReActAgent 中使用
agent = ReActAgent(
    name="assistant",
    model=model,
    memory=InMemoryMemory(),
    long_term_memory=ltm,
    long_term_memory_mode="both"  # 自动检索+记录，Agent 也可主动调用
)
```

---

### 3.6 Formatter 模块

**文件位置**: `src/agentscope/formatter/`

#### 3.6.1 文件结构

```
formatter/
├── __init__.py                      # 导出接口
├── _formatter_base.py              # FormatterBase 基类
├── _truncated_formatter_base.py    # 截断格式化器基类
├── _dashscope_formatter.py         # 通义千问格式化器
├── _openai_formatter.py            # OpenAI 格式化器
├── _anthropic_formatter.py         # Anthropic 格式化器
├── _gemini_formatter.py            # Gemini 格式化器
├── _ollama_formatter.py            # Ollama 格式化器
└── _deepseek_formatter.py          # DeepSeek 格式化器
```

#### 3.6.2 FormatterBase 类

**文件**: `_formatter_base.py`

```python
class FormatterBase(StateModule, ABC):
    """格式化器基类

    功能:
    - 将 Msg 对象转换为 API 要求的格式
    - 处理工具调用和结果
    - 支持多模态内容
    """

    @abstractmethod
    async def format(
        self,
        msgs: list[Msg],
        tools: list[dict] | None = None,
        **kwargs
    ) -> list[dict[str, Any]]:
        """格式化消息列表

        Args:
            msgs: Msg 对象列表
            tools: 工具 JSON Schema 列表

        Returns:
            API 格式的消息列表
        """
```

**辅助方法**:
```python
@staticmethod
def assert_list_of_msgs(msgs: list[Msg]) -> None:
    """验证输入是 Msg 对象列表"""
    if not isinstance(msgs, list):
        raise TypeError("msgs must be a list")

    for msg in msgs:
        if not isinstance(msg, Msg):
            raise TypeError(f"Element must be Msg, got {type(msg)}")

@staticmethod
def convert_tool_result_to_string(
    output: str | List[TextBlock | ImageBlock | AudioBlock]
) -> str:
    """将工具结果转换为文本

    用于不支持多模态工具结果的 API
    """
    if isinstance(output, str):
        return output

    text_blocks = [
        block["text"]
        for block in output
        if block["type"] == "text"
    ]

    return "\n".join(text_blocks)
```

#### 3.6.3 具体格式化器实现

每个模型提供商有两个格式化器:
1. **ChatFormatter**: 单 Agent 对话格式
2. **MultiAgentFormatter**: 多 Agent 对话格式

**1. DashScope 格式化器**

**DashScopeChatFormatter**:
```python
class DashScopeChatFormatter(FormatterBase):
    """通义千问单 Agent 格式化器"""

    async def format(
        self,
        msgs: list[Msg],
        tools: list[dict] | None = None
    ) -> list[dict]:
        """格式化为通义千问 API 格式

        格式:
        {
            "role": "user" | "assistant" | "system",
            "content": [
                {"text": "..."},
                {"image": "url"},
                {"audio": "url"},
                ...
            ]
        }
        """
        self.assert_list_of_msgs(msgs)

        formatted = []

        for msg in msgs:
            formatted_msg = {
                "role": msg.role,
                "content": []
            }

            for block in msg.content:
                if block["type"] == "text":
                    formatted_msg["content"].append({
                        "text": block["text"]
                    })
                elif block["type"] == "image":
                    formatted_msg["content"].append({
                        "image": self._format_image(block)
                    })
                elif block["type"] == "audio":
                    formatted_msg["content"].append({
                        "audio": self._format_audio(block)
                    })
                elif block["type"] == "tool_use":
                    # 工具调用
                    formatted_msg["tool_calls"] = [{
                        "id": block["id"],
                        "type": "function",
                        "function": {
                            "name": block["name"],
                            "arguments": json.dumps(block["input"])
                        }
                    }]
                elif block["type"] == "tool_result":
                    # 工具结果
                    formatted_msg["role"] = "tool"
                    formatted_msg["tool_call_id"] = block["id"]
                    formatted_msg["content"] = block["output"]

            formatted.append(formatted_msg)

        return formatted
```

**DashScopeMultiAgentFormatter**:
```python
class DashScopeMultiAgentFormatter(FormatterBase):
    """通义千问多 Agent 格式化器

    特点:
    - 将 Agent 名称添加到消息中
    - 保留 Agent 身份信息
    """

    async def format(
        self,
        msgs: list[Msg],
        tools: list[dict] | None = None
    ) -> list[dict]:
        """格式化多 Agent 对话"""
        formatted = []

        for msg in msgs:
            # 添加 Agent 名称前缀
            text_content = f"[{msg.name}]: {msg.get_text_content()}"

            formatted_msg = {
                "role": msg.role,
                "content": [{"text": text_content}]
            }

            # ... 处理其他内容块

            formatted.append(formatted_msg)

        return formatted
```

**2. OpenAI 格式化器**

**OpenAIChatFormatter**:
```python
class OpenAIChatFormatter(FormatterBase):
    """OpenAI 单 Agent 格式化器"""

    async def format(
        self,
        msgs: list[Msg],
        tools: list[dict] | None = None
    ) -> list[dict]:
        """格式化为 OpenAI API 格式

        格式:
        {
            "role": "user" | "assistant" | "system",
            "content": "..." | [
                {"type": "text", "text": "..."},
                {"type": "image_url", "image_url": {"url": "..."}},
                ...
            ]
        }
        """
        formatted = []

        for msg in msgs:
            # 处理纯文本消息
            if all(block["type"] == "text" for block in msg.content):
                formatted.append({
                    "role": msg.role,
                    "content": msg.get_text_content()
                })
            else:
                # 多模态消息
                content = []

                for block in msg.content:
                    if block["type"] == "text":
                        content.append({
                            "type": "text",
                            "text": block["text"]
                        })
                    elif block["type"] == "image":
                        content.append({
                            "type": "image_url",
                            "image_url": {
                                "url": self._format_image_url(block)
                            }
                        })
                    # ... 其他类型

                formatted.append({
                    "role": msg.role,
                    "content": content
                })

        return formatted
```

**3. Anthropic 格式化器**

**AnthropicChatFormatter**:
```python
class AnthropicChatFormatter(FormatterBase):
    """Anthropic Claude 格式化器"""

    async def format(
        self,
        msgs: list[Msg],
        tools: list[dict] | None = None
    ) -> list[dict]:
        """格式化为 Anthropic API 格式

        特点:
        - system 消息单独处理
        - 支持 thinking blocks
        - 工具调用格式不同
        """
        formatted = []

        for msg in msgs:
            if msg.role == "system":
                # system 消息单独传递
                continue

            content = []

            for block in msg.content:
                if block["type"] == "text":
                    content.append({
                        "type": "text",
                        "text": block["text"]
                    })
                elif block["type"] == "thinking":
                    # Claude 特有的思考块
                    content.append({
                        "type": "thinking",
                        "thinking": block["thinking"]
                    })
                elif block["type"] == "tool_use":
                    content.append({
                        "type": "tool_use",
                        "id": block["id"],
                        "name": block["name"],
                        "input": block["input"]
                    })
                # ... 其他类型

            formatted.append({
                "role": msg.role,
                "content": content
            })

        return formatted
```

#### 3.6.4 截断格式化器

**TruncatedFormatterBase**:
```python
class TruncatedFormatterBase(FormatterBase):
    """支持截断的格式化器基类

    功能:
    - 限制消息历史长度
    - 基于 token 数量截断
    - 保留重要消息（system、最近消息）
    """

    def __init__(
        self,
        max_tokens: int = 4096,
        token_counter: TokenCounterBase | None = None
    ):
        self.max_tokens = max_tokens
        self.token_counter = token_counter

    async def format(
        self,
        msgs: list[Msg],
        tools: list[dict] | None = None
    ) -> list[dict]:
        """格式化并截断"""
        # 1. 基础格式化
        formatted = await self._base_format(msgs, tools)

        # 2. 计算 tokens
        total_tokens = self.token_counter.count(formatted)

        # 3. 如果超出限制，进行截断
        if total_tokens > self.max_tokens:
            formatted = self._truncate(formatted)

        return formatted

    def _truncate(self, formatted: list[dict]) -> list[dict]:
        """截断策略

        保留:
        - system 消息
        - 最后 N 条消息
        """
        system_msgs = [m for m in formatted if m["role"] == "system"]
        other_msgs = [m for m in formatted if m["role"] != "system"]

        # 从后往前保留，直到不超过限制
        kept = []
        tokens = 0

        for msg in reversed(other_msgs):
            msg_tokens = self.token_counter.count([msg])
            if tokens + msg_tokens <= self.max_tokens:
                kept.insert(0, msg)
                tokens += msg_tokens
            else:
                break

        return system_msgs + kept
```

#### 3.6.5 格式化器使用示例

```python
from agentscope.formatter import DashScopeChatFormatter, OpenAIChatFormatter

# 创建消息
msgs = [
    Msg("system", "You are a helpful assistant", "system"),
    Msg("user", "Hello", "user"),
    Msg("assistant", "Hi! How can I help?", "assistant"),
    Msg("user", "What's 2+2?", "user")
]

# 使用 DashScope 格式化器
dashscope_formatter = DashScopeChatFormatter()
formatted_ds = await dashscope_formatter.format(msgs)

# 使用 OpenAI 格式化器
openai_formatter = OpenAIChatFormatter()
formatted_openai = await openai_formatter.format(msgs)

# 格式化结果可直接传递给对应的模型 API
response = await dashscope_model(messages=formatted_ds)
```

---

继续下一部分...
