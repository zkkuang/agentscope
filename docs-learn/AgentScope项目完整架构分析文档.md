# AgentScope 项目完整架构深度分析文档

> 本文档对 AgentScope 多 Agent 框架进行全面深入的分析，涵盖项目结构、核心模块、实现机制、设计理念和使用指南

**文档版本**: 2.0
**项目版本**: AgentScope 1.0.4
**分析日期**: 2025-10-07
**项目地址**: https://github.com/agentscope-ai/agentscope
**作者**: AgentScope 社区

---

## 📑 目录

- [第一章: 项目概览](#第一章-项目概览)
  - [1.1 项目简介与特色](#11-项目简介与特色)
  - [1.2 技术栈与依赖](#12-技术栈与依赖)
  - [1.3 项目目录结构](#13-项目目录结构)
  - [1.4 初始化与配置](#14-初始化与配置)

- [第二章: 核心架构设计](#第二章-核心架构设计)
  - [2.1 整体架构图](#21-整体架构图)
  - [2.2 核心概念详解](#22-核心概念详解)
  - [2.3 设计哲学](#23-设计哲学)
  - [2.4 异步优先架构](#24-异步优先架构)

- [第三章: Agent 模块深度解析](#第三章-agent-模块深度解析)
  - [3.1 AgentBase 基类](#31-agentbase-基类)
  - [3.2 钩子系统详解](#32-钩子系统详解)
  - [3.3 ReActAgent 实现](#33-reactagent-实现)
  - [3.4 UserAgent 实现](#34-useragent-实现)

- [第四章: Model 模块详细分析](#第四章-model-模块详细分析)
  - [4.1 ChatModelBase 基类](#41-chatmodelbase-基类)
  - [4.2 响应对象设计](#42-响应对象设计)
  - [4.3 各厂商模型实现](#43-各厂商模型实现)
  - [4.4 流式处理机制](#44-流式处理机制)

- [第五章: Message 与通信机制](#第五章-message-与通信机制)
  - [5.1 Msg 类设计](#51-msg-类设计)
  - [5.2 内容块系统](#52-内容块系统)
  - [5.3 多模态支持](#53-多模态支持)
  - [5.4 消息序列化](#54-消息序列化)

- [第六章: Tool 工具系统](#第六章-tool-工具系统)
  - [6.1 Toolkit 管理器](#61-toolkit-管理器)
  - [6.2 工具注册机制](#62-工具注册机制)
  - [6.3 工具组管理](#63-工具组管理)
  - [6.4 MCP 集成](#64-mcp-集成)
  - [6.5 内置工具函数](#65-内置工具函数)

- [第七章: Memory 记忆系统](#第七章-memory-记忆系统)
  - [7.1 短期记忆设计](#71-短期记忆设计)
  - [7.2 长期记忆集成](#72-长期记忆集成)
  - [7.3 记忆管理策略](#73-记忆管理策略)

- [第八章: Formatter 格式化系统](#第八章-formatter-格式化系统)
  - [8.1 格式化器基类](#81-格式化器基类)
  - [8.2 各厂商格式化器](#82-各厂商格式化器)
  - [8.3 截断策略](#83-截断策略)

- [第九章: Pipeline 工作流编排](#第九章-pipeline-工作流编排)
  - [9.1 MsgHub 消息中心](#91-msghub-消息中心)
  - [9.2 Pipeline 模式](#92-pipeline-模式)
  - [9.3 复杂工作流设计](#93-复杂工作流设计)

- [第十章: Plan 规划系统](#第十章-plan-规划系统)
  - [10.1 PlanNotebook 设计](#101-plannotebook-设计)
  - [10.2 任务分解机制](#102-任务分解机制)
  - [10.3 计划执行追踪](#103-计划执行追踪)

- [第十一章: RAG 检索增强](#第十一章-rag-检索增强)
  - [11.1 KnowledgeBase 设计](#111-knowledgebase-设计)
  - [11.2 文档处理](#112-文档处理)
  - [11.3 向量存储](#113-向量存储)

- [第十二章: MCP 协议集成](#第十二章-mcp-协议集成)
  - [12.1 MCP 客户端架构](#121-mcp-客户端架构)
  - [12.2 HTTP 客户端](#122-http-客户端)
  - [12.3 StdIO 客户端](#123-stdio-客户端)

- [第十三章: 辅助模块](#第十三章-辅助模块)
  - [13.1 Embedding 嵌入模型](#131-embedding-嵌入模型)
  - [13.2 Token 计数](#132-token-计数)
  - [13.3 Session 会话管理](#133-session-会话管理)
  - [13.4 Tracing 追踪系统](#134-tracing-追踪系统)

- [第十四章: 执行流程深度剖析](#第十四章-执行流程深度剖析)
  - [14.1 ReActAgent 完整执行流](#141-reactagent-完整执行流)
  - [14.2 多 Agent 协作流程](#142-多-agent-协作流程)
  - [14.3 工具调用流程](#143-工具调用流程)

- [第十五章: 扩展开发指南](#第十五章-扩展开发指南)
  - [15.1 自定义 Agent](#151-自定义-agent)
  - [15.2 自定义工具函数](#152-自定义工具函数)
  - [15.3 自定义组件](#153-自定义组件)

- [第十六章: 最佳实践与优化](#第十六章-最佳实践与优化)
  - [16.1 性能优化](#161-性能优化)
  - [16.2 错误处理](#162-错误处理)
  - [16.3 测试策略](#163-测试策略)

---

## 第一章: 项目概览

### 1.1 项目简介与特色

#### 1.1.1 项目背景

**AgentScope** 是由阿里巴巴通义实验室 SysML 团队开发的开源多 Agent 框架。该项目始于 2024 年初，目标是为 LLM 应用开发者提供一个面向 Agent 的编程范式，使复杂的多 Agent 系统开发变得简单、透明、可控。

#### 1.1.2 核心设计理念

AgentScope 的设计遵循以下核心理念：

1. **透明性优先 (Transparency First)**
   - **理念**: 开发者应该清楚地知道系统内部发生了什么
   - **实现**: 所有操作都是可见的，没有深度封装的"黑盒"
   - **好处**: 易于调试、理解和优化
   - **示例**: Agent 的每次推理、工具调用都有明确的日志和追踪

2. **异步优先 (Async First)**
   - **理念**: 现代 AI 应用需要处理大量并发请求
   - **实现**: 所有核心方法都是异步的(`async/await`)
   - **好处**: 高并发性能、非阻塞执行、支持流式输出
   - **示例**:
     ```python
     # 并行执行多个 Agent
     results = await asyncio.gather(
         agent1(msg),
         agent2(msg),
         agent3(msg)
     )
     ```

3. **高度模块化 (Highly Modular)**
   - **理念**: 系统应该像乐高积木一样可组合
   - **实现**: 清晰的模块边界，每个组件都可独立替换
   - **好处**: 易于扩展、定制和测试
   - **示例**: 可以轻松替换 Model、Memory、Formatter 等组件

4. **多 Agent 导向 (Multi-Agent Oriented)**
   - **理念**: 复杂任务需要多个 Agent 协作
   - **实现**: 显式消息传递、MsgHub 发布-订阅、Pipeline 编排
   - **好处**: 支持复杂的协作模式
   - **示例**:
     ```python
     # 三个专家 Agent 讨论一个问题
     async with MsgHub([expert1, expert2, expert3]) as hub:
         await sequential_pipeline([expert1, expert2, expert3])
     ```

5. **模型无关 (Model Agnostic)**
   - **理念**: 不绑定特定的 LLM 提供商
   - **实现**: 统一的模型接口，支持多种 API
   - **好处**: 灵活选择最适合的模型
   - **支持**: OpenAI、Anthropic、阿里云、Google、Ollama 等

6. **实时控制 (Real-time Control)**
   - **理念**: 开发者应该能够实时干预 Agent 行为
   - **实现**: 钩子系统、中断机制
   - **好处**: 可以在执行过程中修改行为
   - **示例**: 在工具调用前检查并修改参数

#### 1.1.3 核心特性详解

**✨ 透明性 (Transparency)**

```python
# 所有操作都是可见的
class MyAgent(ReActAgent):
    async def reply(self, msg):
        logger.info(f"Agent {self.name} 开始处理消息: {msg}")

        # 推理过程可见
        reasoning_result = await self._reasoning()
        logger.info(f"推理结果: {reasoning_result}")

        # 工具调用可见
        if tool_calls := reasoning_result.get_content_blocks("tool_use"):
            logger.info(f"准备调用 {len(tool_calls)} 个工具")

        return await super().reply(msg)
```

**⚡ 异步性能 (Asynchronous Performance)**

```python
# 并行工具调用
async def _acting(self, tool_calls):
    # 多个工具并行执行，而不是顺序执行
    results = await asyncio.gather(*[
        self.toolkit.call_tool_function(tc)
        for tc in tool_calls
    ])
    return results

# 流式输出
async for chunk in agent.reply_stream(msg):
    print(chunk, end="", flush=True)
```

**🔧 模块化设计 (Modularity)**

```python
# 每个组件都可以独立配置和替换
agent = ReActAgent(
    name="assistant",
    model=DashScopeChatModel(...),      # 可替换为 OpenAI、Anthropic 等
    memory=InMemoryMemory(),            # 可替换为 Redis、数据库等
    formatter=DashScopeChatFormatter(), # 可替换为其他格式化器
    toolkit=Toolkit(),                  # 可自定义工具集
    long_term_memory=Mem0LTM(...),     # 可选的长期记忆
    knowledge=[kb1, kb2],               # 可选的知识库
    plan_notebook=PlanNotebook()        # 可选的规划系统
)
```

**🤖 多 Agent 协作 (Multi-Agent Collaboration)**

AgentScope 提供三种主要的多 Agent 协作模式：

1. **顺序执行 (Sequential)**
   ```python
   # Agent 依次处理，前一个的输出作为下一个的输入
   result = await sequential_pipeline([
       preprocessor_agent,
       analyzer_agent,
       summarizer_agent
   ], initial_msg)
   ```

2. **并行执行 (Parallel/Fanout)**
   ```python
   # 所有 Agent 并行处理相同输入
   results = await fanout_pipeline([
       expert_a, expert_b, expert_c
   ], query_msg)
   ```

3. **消息中心 (Message Hub)**
   ```python
   # Agent 之间自动广播消息
   async with MsgHub([agent1, agent2, agent3], announcement=topic_msg) as hub:
       # 任何 Agent 的输出都会自动广播给其他 Agent
       await agent1(msg)  # 输出会自动发给 agent2 和 agent3
       await agent2()      # agent2 已经"听到"了 agent1 的回复
   ```

**🛠️ 丰富的工具生态 (Rich Tool Ecosystem)**

- **内置工具**: Python 代码执行、Shell 命令、文件操作
- **MCP 协议**: 连接外部工具服务器
- **自定义工具**: 简单的装饰器即可注册
- **工具组管理**: 动态激活/停用工具组

```python
# 简单注册自定义工具
@toolkit.register_tool_function
def my_calculator(a: int, b: int) -> int:
    """计算两个数的和

    Args:
        a: 第一个数
        b: 第二个数
    """
    return a + b

# MCP 工具集成
await toolkit.register_mcp_client(mcp_client)
```

**🎯 实时控制与中断 (Real-time Control)**

```python
# 钩子函数允许在任何环节插入自定义逻辑
def check_and_modify_tools(self, kwargs):
    """在工具调用前检查参数"""
    tool_calls = kwargs.get("tool_calls", [])

    for tc in tool_calls:
        if tc["name"] == "dangerous_operation":
            # 阻止危险操作
            raise ValueError("此操作需要人工确认")

    return kwargs

# 注册钩子
ReActAgent.register_class_hook(
    "pre_acting",
    "safety_check",
    check_and_modify_tools
)
```

#### 1.1.4 应用场景

AgentScope 适用于以下场景：

**单 Agent 应用**:
- 智能客服机器人
- 代码助手
- 内容生成工具
- 数据分析助手
- 个人助理

**多 Agent 应用**:
- 团队协作模拟（如会议讨论）
- 多角色游戏（如狼人杀）
- 复杂决策系统
- 研究团队模拟
- 辩论系统

**企业级应用**:
- 智能客服系统
- 知识管理平台
- 自动化工作流
- 文档处理系统
- 研究助手

---

### 1.2 技术栈与依赖

#### 1.2.1 核心技术栈

**Python 版本要求**: Python 3.10+

AgentScope 选择 Python 3.10+ 的原因：
- **类型注解增强**: `X | Y` 语法、`TypedDict` 改进
- **模式匹配**: `match-case` 语句
- **更好的错误消息**: 提升开发体验
- **性能优化**: CPython 性能提升

**核心依赖库**:

```python
# 异步编程
aioitertools>=0.11.0    # 异步迭代工具
asyncio                 # Python 内置异步库

# LLM API 客户端
anthropic>=0.36.0       # Anthropic Claude API
dashscope>=1.20.0       # 阿里云通义千问 API
openai>=1.47.0          # OpenAI API
google-genai            # Google Gemini API (可选)
ollama                  # Ollama 本地模型 (可选)

# MCP 协议
mcp>=1.0.0             # Model Context Protocol

# 追踪与监控
opentelemetry-api      # 分布式追踪 API
opentelemetry-sdk      # 追踪 SDK
opentelemetry-exporter-otlp  # OTLP 导出器

# Token 处理
tiktoken>=0.8.0        # OpenAI Token 计数器

# 数据验证
pydantic>=2.0.0        # 数据验证库

# 其他
shortuuid              # 生成短 UUID
docstring_parser       # 解析函数文档字符串
```

**可选依赖**:

```python
# RAG 相关
pypdf>=4.0.0           # PDF 文件处理
nltk>=3.8.0            # 自然语言处理
qdrant-client>=1.7.0   # Qdrant 向量数据库客户端

# 长期记忆
mem0ai>=0.1.0          # Mem0 记忆平台

# 评估
ray>=2.0.0             # 分布式计算

# HuggingFace 模型
transformers>=4.0.0    # Transformer 模型库
torch>=2.0.0           # PyTorch

# Web 工具
beautifulsoup4         # HTML 解析
requests               # HTTP 请求
```

#### 1.2.2 依赖安装

**基础安装**:
```bash
pip install agentscope
```

**完整安装**:
```bash
pip install "agentscope[full]"
```

**按需安装**:
```bash
# RAG 功能
pip install "agentscope[rag]"

# 长期记忆
pip install "agentscope[mem0]"

# 评估功能
pip install "agentscope[eval]"

# HuggingFace 支持
pip install "agentscope[huggingface]"

# 开发工具
pip install "agentscope[dev]"
```

#### 1.2.3 架构层次

AgentScope 采用分层架构：

```
┌─────────────────────────────────────────┐
│      应用层 (Application Layer)          │
│  用户自定义 Agent、工具、工作流           │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│      框架层 (Framework Layer)            │
│  Agent、Pipeline、Plan、RAG             │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│      核心层 (Core Layer)                 │
│  Model、Memory、Tool、Formatter         │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│      基础层 (Foundation Layer)           │
│  Message、StateModule、Utils            │
└─────────────────────────────────────────┘
```

---

### 1.3 项目目录结构

#### 1.3.1 完整目录树

```
agentscope/
├── src/agentscope/              # 核心源码目录
│   ├── __init__.py             # 包初始化，导出主要 API
│   ├── _logging.py             # 日志配置
│   ├── _constants.py           # 常量定义
│   │
│   ├── agent/                  # Agent 实现
│   │   ├── __init__.py
│   │   ├── _agent_base.py      # AgentBase 基类 (300+ 行)
│   │   ├── _agent_meta.py      # Agent 元类，处理钩子继承
│   │   ├── _react_agent_base.py # ReActAgentBase 基类
│   │   ├── _react_agent.py     # ReActAgent 完整实现 (500+ 行)
│   │   ├── _user_agent.py      # UserAgent 用户交互
│   │   └── _user_input.py      # 用户输入处理器
│   │
│   ├── model/                  # 模型接口
│   │   ├── __init__.py
│   │   ├── _model_base.py      # ChatModelBase 基类
│   │   ├── _model_response.py  # ChatResponse 响应类
│   │   ├── _model_usage.py     # 使用统计
│   │   ├── _dashscope_model.py # 阿里云通义千问
│   │   ├── _openai_model.py    # OpenAI GPT 系列
│   │   ├── _anthropic_model.py # Anthropic Claude
│   │   ├── _ollama_model.py    # Ollama 本地模型
│   │   ├── _gemini_model.py    # Google Gemini
│   │   └── _huggingface_model.py # HuggingFace 模型
│   │
│   ├── message/                # 消息系统
│   │   ├── __init__.py
│   │   ├── _message_base.py    # Msg 类 (200+ 行)
│   │   └── _message_block.py   # 内容块定义 (TextBlock, ImageBlock 等)
│   │
│   ├── tool/                   # 工具系统
│   │   ├── __init__.py
│   │   ├── _toolkit.py         # Toolkit 管理器 (600+ 行)
│   │   ├── _response.py        # ToolResponse 类
│   │   ├── _registered_tool_function.py  # 已注册工具包装
│   │   ├── _async_wrapper.py   # 异步包装器
│   │   ├── _coding/            # 代码执行工具
│   │   │   ├── _python.py      # Python 代码执行
│   │   │   └── _shell.py       # Shell 命令执行
│   │   ├── _text_file/         # 文件操作工具
│   │   │   ├── _view_text_file.py
│   │   │   ├── _write_text_file.py
│   │   │   ├── _insert_text_file.py
│   │   │   └── _utils.py
│   │   └── _multi_modality/    # 多模态工具
│   │       ├── _dashscope_tools.py  # 通义万相、听悟等
│   │       └── _openai_tools.py     # DALL-E、Whisper 等
│   │
│   ├── formatter/              # 提示词格式化
│   │   ├── __init__.py
│   │   ├── _formatter_base.py  # FormatterBase 基类
│   │   ├── _truncated_formatter_base.py  # 截断格式化器
│   │   ├── _dashscope_formatter.py  # 通义千问格式
│   │   ├── _openai_formatter.py     # OpenAI 格式
│   │   ├── _anthropic_formatter.py  # Anthropic 格式
│   │   ├── _gemini_formatter.py     # Gemini 格式
│   │   ├── _ollama_formatter.py     # Ollama 格式
│   │   └── _deepseek_formatter.py   # DeepSeek 格式
│   │
│   ├── memory/                 # 记忆系统
│   │   ├── __init__.py
│   │   ├── _memory_base.py     # MemoryBase 基类
│   │   ├── _in_memory_memory.py # 内存记忆实现
│   │   ├── _long_term_memory_base.py  # 长期记忆基类
│   │   ├── _mem0_long_term_memory.py  # Mem0 集成
│   │   └── _mem0_utils.py      # Mem0 工具函数
│   │
│   ├── pipeline/               # 工作流编排
│   │   ├── __init__.py
│   │   ├── _msghub.py          # MsgHub 消息中心 (200+ 行)
│   │   ├── _class.py           # Pipeline 类 (Sequential、Fanout)
│   │   └── _functional.py      # 函数式 Pipeline
│   │
│   ├── plan/                   # 规划系统
│   │   ├── __init__.py
│   │   ├── _plan_model.py      # Plan 和 SubTask 数据模型
│   │   ├── _plan_notebook.py   # PlanNotebook 主类 (400+ 行)
│   │   ├── _storage_base.py    # PlanStorageBase 存储基类
│   │   └── _in_memory_storage.py  # 内存存储实现
│   │
│   ├── rag/                    # RAG 检索增强
│   │   ├── __init__.py
│   │   ├── _document.py        # Document 文档模型
│   │   ├── _knowledge_base.py  # KnowledgeBase 基类
│   │   ├── _simple_knowledge.py # SimpleKnowledgeBase 实现
│   │   ├── _reader/            # 文档读取器
│   │   │   ├── _reader_base.py
│   │   │   ├── _text_reader.py  # 文本文件读取
│   │   │   ├── _pdf_reader.py   # PDF 读取
│   │   │   └── _image_reader.py # 图片读取
│   │   └── _store/             # 向量存储
│   │       ├── _store_base.py
│   │       └── _qdrant_store.py  # Qdrant 向量数据库
│   │
│   ├── mcp/                    # MCP 协议
│   │   ├── __init__.py
│   │   ├── _client_base.py     # MCPClientBase 基类
│   │   ├── _mcp_function.py    # MCP 工具函数包装
│   │   ├── _stateful_client_base.py  # 有状态客户端基类
│   │   ├── _stdio_stateful_client.py # StdIO 客户端
│   │   ├── _http_stateless_client.py # HTTP 无状态客户端
│   │   └── _http_stateful_client.py  # HTTP 有状态客户端
│   │
│   ├── session/                # 会话管理
│   │   ├── __init__.py
│   │   ├── _session_base.py    # SessionBase 基类
│   │   └── _json_session.py    # JSON 文件会话
│   │
│   ├── embedding/              # 嵌入模型
│   │   ├── __init__.py
│   │   ├── _embedding_base.py  # EmbeddingModelBase
│   │   ├── _dashscope_embedding.py  # 通义嵌入
│   │   ├── _openai_embedding.py     # OpenAI 嵌入
│   │   └── _cache.py           # 嵌入缓存
│   │
│   ├── token/                  # Token 处理
│   │   ├── __init__.py
│   │   ├── _token_counter_base.py  # Token 计数器基类
│   │   ├── _openai_counter.py      # OpenAI 计数器
│   │   ├── _anthropic_counter.py   # Anthropic 计数器
│   │   └── _gemini_counter.py      # Gemini 计数器
│   │
│   ├── evaluate/               # 评估框架
│   │   ├── __init__.py
│   │   ├── _evaluator.py       # Evaluator 评估器
│   │   ├── _task.py            # Task 任务定义
│   │   ├── _solution.py        # Solution 解决方案
│   │   └── _ray_evaluator.py   # 分布式评估
│   │
│   ├── tracing/                # 追踪系统
│   │   ├── __init__.py
│   │   ├── _trace.py           # 追踪装饰器
│   │   ├── _setup.py           # 追踪设置
│   │   └── _exporter.py        # 追踪导出器
│   │
│   ├── hooks/                  # 钩子系统
│   │   ├── __init__.py
│   │   ├── _studio_hooks.py    # AgentScope Studio 钩子
│   │   └── _logging_hooks.py   # 日志钩子
│   │
│   ├── module/                 # 基础模块
│   │   ├── __init__.py
│   │   └── _state_module.py    # StateModule 状态管理基类
│   │
│   ├── exception/              # 异常定义
│   │   ├── __init__.py
│   │   └── _exception.py       # 各种异常类
│   │
│   ├── types/                  # 类型定义
│   │   ├── __init__.py
│   │   └── _types.py           # TypedDict 和 Protocol 定义
│   │
│   └── _utils/                 # 工具函数
│       ├── _common.py          # 通用工具
│       ├── _json_schema.py     # JSON Schema 生成
│       └── _docstring.py       # 文档字符串解析
│
├── examples/                   # 示例代码
│   ├── conversation/           # 对话示例
│   ├── game/                   # 游戏示例 (狼人杀等)
│   ├── pipeline/               # Pipeline 示例
│   ├── rag/                    # RAG 示例
│   └── plan/                   # 规划示例
│
├── tests/                      # 测试代码
│   ├── agent/
│   ├── model/
│   ├── tool/
│   └── ...
│
├── docs/                       # 文档
│   ├── sphinx/                 # Sphinx 文档源码
│   ├── tutorial/               # 教程
│   └── api/                    # API 文档
│
├── scripts/                    # 辅助脚本
│   ├── install/                # 安装脚本
│   └── lint/                   # 代码检查脚本
│
├── setup.py                    # 安装配置
├── pyproject.toml              # 项目配置
├── README.md                   # 项目说明
├── LICENSE                     # 许可证 (Apache 2.0)
└── .gitignore                  # Git 忽略文件
```

#### 1.3.2 模块组织原则

AgentScope 的目录组织遵循以下原则：

1. **单一职责**: 每个目录/文件只负责一个功能模块
2. **分层清晰**: 基础层 → 核心层 → 框架层 → 应用层
3. **依赖方向**: 高层依赖低层，低层不依赖高层
4. **命名规范**:
   - 私有模块以 `_` 开头 (如 `_agent_base.py`)
   - 公共 API 在 `__init__.py` 中导出
   - 类名使用大驼峰 (如 `ReActAgent`)
   - 函数名使用小写下划线 (如 `register_tool_function`)

---

### 1.4 初始化与配置

#### 1.4.1 初始化函数

**文件位置**: `src/agentscope/__init__.py`

```python
def init(
    project: str | None = None,
    name: str | None = None,
    logging_path: str | None = None,
    logging_level: str = "INFO",
    studio_url: str | None = None,
    tracing_url: str | None = None,
    save_code: bool = False,
    save_api_invoke: bool = False,
    use_monitor: bool = True,
    cache_dir: str | None = None,
) -> None:
    """初始化 AgentScope 框架

    Args:
        project: 项目名称，用于组织日志和追踪数据
        name: 运行名称，标识一次运行实例
        logging_path: 日志文件保存路径
        logging_level: 日志级别 (DEBUG, INFO, WARNING, ERROR, CRITICAL)
        studio_url: AgentScope Studio URL，用于可视化和交互
        tracing_url: OpenTelemetry 追踪端点 URL
        save_code: 是否保存代码快照
        save_api_invoke: 是否保存 API 调用记录
        use_monitor: 是否启用资源监控
        cache_dir: 缓存目录（用于 Embedding 等）
    """
```

**设计理念**:

1. **全局配置**: 一次初始化，全局生效
2. **可选参数**: 所有参数都是可选的，有合理的默认值
3. **灵活性**: 可以在运行时动态调整某些配置

**内部实现**:

```python
def init(
    project: str | None = None,
    name: str | None = None,
    **kwargs
) -> None:
    # 1. 设置项目和运行名称
    _runtime.project = project or "default_project"
    _runtime.name = name or f"run_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

    # 2. 配置日志系统
    _setup_logging(
        logging_path=kwargs.get("logging_path"),
        logging_level=kwargs.get("logging_level", "INFO")
    )

    # 3. 初始化追踪系统
    if tracing_url := kwargs.get("tracing_url"):
        from agentscope.tracing import setup_tracing
        setup_tracing(
            endpoint=tracing_url,
            service_name=_runtime.project
        )

    # 4. 连接 Studio (如果提供)
    if studio_url := kwargs.get("studio_url"):
        from agentscope.hooks import setup_studio_hooks
        setup_studio_hooks(studio_url)

    # 5. 创建缓存目录
    if cache_dir := kwargs.get("cache_dir"):
        os.makedirs(cache_dir, exist_ok=True)
        _runtime.cache_dir = cache_dir

    logger.info(f"AgentScope 初始化完成: project={_runtime.project}, name={_runtime.name}")
```

#### 1.4.2 使用示例

**最简单的初始化**:
```python
import agentscope

# 使用默认配置
agentscope.init()
```

**开发环境初始化**:
```python
import agentscope

agentscope.init(
    project="my_chatbot",
    name="debug_run_001",
    logging_level="DEBUG",
    logging_path="./logs"
)
```

**生产环境初始化**:
```python
import agentscope

agentscope.init(
    project="production_chatbot",
    name=f"prod_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
    logging_level="WARNING",
    logging_path="/var/log/agentscope",
    tracing_url="http://jaeger:4317",  # Jaeger 追踪
    save_api_invoke=True,               # 保存 API 调用
    cache_dir="/var/cache/agentscope"
)
```

**与 Studio 集成**:
```python
import agentscope

agentscope.init(
    project="interactive_demo",
    name="studio_session_001",
    studio_url="http://localhost:5000",  # Studio 地址
    logging_level="INFO"
)

# 之后所有 UserAgent 的输入都会从 Studio 获取
# 所有 Agent 的输出都会发送到 Studio 显示
```

#### 1.4.3 环境变量配置

AgentScope 还支持通过环境变量进行配置：

```bash
# API Keys
export DASHSCOPE_API_KEY="your_dashscope_key"
export OPENAI_API_KEY="your_openai_key"
export ANTHROPIC_API_KEY="your_anthropic_key"

# 项目配置
export AGENTSCOPE_PROJECT="my_project"
export AGENTSCOPE_LOGGING_LEVEL="INFO"
export AGENTSCOPE_CACHE_DIR="/tmp/agentscope_cache"

# Studio 配置
export AGENTSCOPE_STUDIO_URL="http://localhost:5000"

# 追踪配置
export AGENTSCOPE_TRACING_URL="http://localhost:4317"
```

使用环境变量后，初始化可以更简单：

```python
import agentscope

# 自动读取环境变量
agentscope.init()
```

---

## 第二章: 核心架构设计

### 2.1 整体架构图

AgentScope 采用分层架构，每一层都有明确的职责：

```
┌────────────────────────────────────────────────────────────────┐
│                     应用层 (Application)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ 自定义 Agent  │  │ 自定义工具    │  │ 工作流编排    │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
└────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────────┐
│                     框架层 (Framework)                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ Pipeline │  │   Plan   │  │   RAG    │  │   MCP    │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
└────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────────┐
│                     核心层 (Core)                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │  Agent   │  │  Model   │  │   Tool   │  │  Memory  │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │Formatter │  │ Embedding│  │  Token   │  │  Tracing │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
└────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────────┐
│                     基础层 (Foundation)                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ Message  │  │StateModule│ │Exception │  │   Utils  │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
└────────────────────────────────────────────────────────────────┘
```

#### 2.1.1 各层职责

**基础层 (Foundation Layer)**:
- **Message**: 定义消息结构和内容块
- **StateModule**: 提供状态序列化/反序列化能力
- **Exception**: 定义异常体系
- **Utils**: 提供通用工具函数

**核心层 (Core Layer)**:
- **Agent**: Agent 的基础实现
- **Model**: LLM 接口抽象
- **Tool**: 工具管理和执行
- **Memory**: 记忆管理
- **Formatter**: 消息格式化
- **Embedding**: 向量嵌入
- **Token**: Token 计数
- **Tracing**: 追踪和监控

**框架层 (Framework Layer)**:
- **Pipeline**: 多 Agent 工作流编排
- **Plan**: 任务规划和分解
- **RAG**: 检索增强生成
- **MCP**: Model Context Protocol 集成

**应用层 (Application Layer)**:
- 用户自定义的 Agent
- 用户自定义的工具
- 业务逻辑和工作流

#### 2.1.2 数据流图

```
┌──────────┐
│   User   │
│  Input   │
└─────┬────┘
      │
      ▼
┌──────────────┐
│  UserAgent   │  (1) 获取用户输入
└─────┬────────┘
      │ Msg
      ▼
┌───────────────────────────────────────────────────────────┐
│                    ReActAgent                              │
│                                                            │
│  (2) observe(msg)                                         │
│      └─► memory.add(msg)                                  │
│                                                            │
│  (3) reply()                                              │
│      ├─► long_term_memory.retrieve() [可选]               │
│      │    └─► 从 Mem0 检索历史知识                         │
│      │                                                     │
│      ├─► knowledge.retrieve() [可选]                      │
│      │    ├─► embedding_model(query)                      │
│      │    └─► vector_store.search()                       │
│      │                                                     │
│      ├─► plan_notebook.get_hint() [可选]                  │
│      │    └─► 获取当前子任务提示                            │
│      │                                                     │
│      └─► ReAct 循环 (最多 max_iters 次)                   │
│           ├─► _reasoning()                                │
│           │    ├─► memory.get_memory()                    │
│           │    ├─► formatter.format(msgs, tools)          │
│           │    ├─► model(formatted_msgs)                  │
│           │    └─► ChatResponse → Msg                     │
│           │                                                │
│           ├─► 提取 tool_calls                             │
│           │                                                │
│           ├─► _acting(tool_calls) [如果有工具调用]        │
│           │    └─► toolkit.call_tool_function()           │
│           │         ├─► 查找工具函数                       │
│           │         ├─► 执行 (可能并行)                    │
│           │         └─► ToolResponse                      │
│           │                                                │
│           ├─► memory.add(tool_results)                    │
│           │                                                │
│           └─► 如果没有工具调用，退出循环                   │
│                                                            │
│  (4) long_term_memory.record() [可选]                     │
│                                                            │
│  (5) broadcast_to_subscribers()                           │
│                                                            │
│  (6) print(response_msg)                                  │
│                                                            │
└───────────────────────┬───────────────────────────────────┘
                        │ Msg (Response)
                        ▼
                  ┌──────────┐
                  │   User   │
                  │  Output  │
                  └──────────┘
```

#### 2.1.3 模块依赖关系

```
┌─────────────────────────────────────────────────────────────┐
│                    依赖关系图                                │
└─────────────────────────────────────────────────────────────┘

              AgentBase
                 │
                 ├──depends──► Message
                 ├──depends──► StateModule
                 ├──depends──► MemoryBase
                 └──uses────► Hooks
                      │
         ┌────────────┼────────────┐
         │            │            │
   ReActAgentBase  UserAgent   CustomAgent
         │
         ├──depends──► ChatModelBase
         ├──depends──► FormatterBase
         ├──depends──► Toolkit
         ├──depends──► LongTermMemoryBase (可选)
         ├──depends──► KnowledgeBase (可选)
         └──depends──► PlanNotebook (可选)
                │
           ReActAgent
                │
                └──used by──► Pipeline, Plan, etc.
```

**依赖规则**:
1. **单向依赖**: 高层模块依赖低层模块，反之不成立
2. **接口依赖**: 依赖抽象(基类)而非具体实现
3. **可选依赖**: 高级功能(RAG、Plan等)是可选的
4. **循环避免**: 没有循环依赖

---

### 2.2 核心概念详解

#### 2.2.1 Agent (智能体)

**定义**: Agent 是一个能够感知环境、自主决策并采取行动的实体。

**核心特征**:
1. **自主性 (Autonomy)**: 可以独立运行，无需人工干预
2. **反应性 (Reactivity)**: 能感知环境变化并做出响应
3. **主动性 (Proactivity)**: 能主动采取行动达成目标
4. **社交性 (Social Ability)**: 能与其他 Agent 通信协作

**在 AgentScope 中的实现**:

```python
class AgentBase(StateModule):
    """Agent 基类

    核心能力:
    1. observe(msg): 感知 - 接收消息但不回复
    2. reply(msg): 决策 - 处理消息并生成回复
    3. __call__(msg): 执行 - 完整的感知-决策-行动循环
    4. print(msg): 行动 - 输出消息给用户或其他 Agent
    """

    async def observe(self, msg: Msg) -> None:
        """感知环境 - 接收信息但不做响应"""
        await self.memory.add(msg)

    async def reply(self, msg: Msg) -> Msg:
        """决策过程 - 核心逻辑,由子类实现"""
        raise NotImplementedError

    async def __call__(self, msg: Msg) -> Msg:
        """完整循环 - 感知、决策、行动"""
        # 1. 感知
        await self.observe(msg)

        # 2. 决策
        response = await self.reply(msg)

        # 3. 行动
        await self.print(response)

        # 4. 社交 (广播给订阅者)
        await self._broadcast_to_subscribers(response)

        return response
```

**Agent 类型**:

1. **ReActAgent**: 使用 Reasoning + Acting 模式
   - 循环执行：思考 → 行动 → 观察结果 → 思考...
   - 适用于需要使用工具的复杂任务

2. **UserAgent**: 与人类用户交互
   - 从终端、Studio 或自定义界面获取输入
   - 适用于需要人工参与的场景

3. **自定义 Agent**: 继承 AgentBase 实现特定逻辑
   - 例如: 只做总结的 SummaryAgent
   - 例如: 专门做分类的 ClassifierAgent

#### 2.2.2 Message (消息)

**定义**: Message 是 Agent 之间通信的基本单位，包含内容、发送者、角色等信息。

**设计理念**:
1. **统一接口**: 所有 Agent 使用相同的消息格式
2. **多模态**: 支持文本、图片、音频、视频等多种内容类型
3. **可序列化**: 可以保存和恢复
4. **可扩展**: 可以添加自定义元数据

**消息结构**:

```python
@dataclass
class Msg:
    """消息类

    属性:
        id: 消息唯一标识
        name: 发送者名称
        content: 内容(文本或内容块列表)
        role: 角色(user/assistant/system)
        metadata: 元数据(可选)
        timestamp: 时间戳
        invocation_id: 调用标识(同一次调用的消息共享)
    """
    id: str
    name: str
    content: str | Sequence[ContentBlock]
    role: Literal["user", "assistant", "system"]
    metadata: dict | None
    timestamp: str
    invocation_id: str
```

**内容块 (Content Blocks)**:

AgentScope 支持多种内容块，实现真正的多模态通信：

```python
# 文本块
TextBlock(type="text", text="Hello, world!")

# 图片块
ImageBlock(
    type="image",
    source=URLSource(type="url", url="https://example.com/image.jpg")
)

# 音频块
AudioBlock(
    type="audio",
    source=Base64Source(
        type="base64",
        media_type="audio/wav",
        data="base64_encoded_data..."
    )
)

# 工具调用块
ToolUseBlock(
    type="tool_use",
    id="call_123",
    name="execute_python_code",
    input={"code": "print('hello')"}
)

# 工具结果块
ToolResultBlock(
    type="tool_result",
    id="call_123",
    output="hello\n"
)

# 思考块 (Claude 等模型支持)
ThinkingBlock(
    type="thinking",
    thinking="Let me analyze this step by step..."
)
```

**消息传递模式**:

1. **点对点 (Point-to-Point)**
   ```python
   # Agent A 向 Agent B 发送消息
   response = await agent_b(msg)
   ```

2. **发布-订阅 (Pub-Sub)**
   ```python
   # 通过 MsgHub, Agent A 的输出自动发给 B 和 C
   async with MsgHub([agent_a, agent_b, agent_c]):
       await agent_a(msg)  # B 和 C 自动收到
   ```

3. **广播 (Broadcast)**
   ```python
   # 手动广播给所有订阅者
   await hub.broadcast(msg)
   ```

#### 2.2.3 Tool (工具)

**定义**: Tool 是 Agent 可以调用的外部功能，扩展 Agent 的能力边界。

**工具哲学**:
- **能力扩展**: LLM 本身无法执行代码、访问网络、操作文件，工具弥补这些能力
- **可靠性**: 工具执行确定性操作，避免 LLM 幻觉
- **可控性**: 工具执行可以被监控、限制和审计

**工具类型**:

1. **内置工具**: AgentScope 提供的工具
   ```python
   execute_python_code(code="print(2+2)")
   view_text_file(file_path="data.txt")
   dashscope_text_to_image(prompt="a cat")
   ```

2. **自定义工具**: 用户自己编写的工具
   ```python
   def my_calculator(a: int, b: int) -> int:
       """计算两数之和

       Args:
           a: 第一个数
           b: 第二个数
       """
       return a + b

   toolkit.register_tool_function(my_calculator)
   ```

3. **MCP 工具**: 通过 MCP 协议连接的外部工具
   ```python
   mcp_client = HttpStatelessClient(url="https://mcp.example.com")
   await toolkit.register_mcp_client(mcp_client)
   ```

**工具调用流程**:

```
1. LLM 决定调用工具
   └─► ChatResponse 包含 ToolUseBlock

2. Agent 提取工具调用
   └─► tool_calls = msg.get_content_blocks("tool_use")

3. Toolkit 执行工具
   ├─► 查找工具函数
   ├─► 验证参数
   ├─► 执行函数 (可能并行)
   └─► 返回 ToolResponse

4. 将结果添加到记忆
   └─► memory.add(ToolResultBlock(...))

5. LLM 基于结果继续推理
   └─► 下一轮 ReAct 循环
```

#### 2.2.4 Memory (记忆)

**定义**: Memory 存储 Agent 的对话历史和知识，使 Agent 能够保持上下文。

**记忆分类**:

1. **短期记忆 (Short-term Memory)**
   - **用途**: 存储当前对话历史
   - **实现**: InMemoryMemory (内存列表)
   - **特点**: 快速、易失性
   - **生命周期**: 对话期间

2. **长期记忆 (Long-term Memory)**
   - **用途**: 存储持久化知识
   - **实现**: Mem0LongTermMemory (基于 Mem0 平台)
   - **特点**: 持久化、可检索
   - **生命周期**: 跨会话

**记忆操作**:

```python
# 短期记忆
memory = InMemoryMemory()
await memory.add(msg)                    # 添加
messages = await memory.get_memory()     # 获取
await memory.clear()                     # 清空

# 长期记忆
ltm = Mem0LongTermMemory(api_key="...")
await ltm.record(messages, agent_id="agent_001")  # 记录
retrieved = await ltm.retrieve(query="...", agent_id="agent_001")  # 检索
```

**记忆管理策略**:

AgentScope 提供三种长期记忆管理模式：

1. **agent_control**: Agent 自主控制
   ```python
   agent = ReActAgent(
       long_term_memory=ltm,
       long_term_memory_mode="agent_control"
   )
   # Agent 可以调用 retrieve_from_memory 和 record_to_memory 工具
   ```

2. **static_control**: 自动管理
   ```python
   agent = ReActAgent(
       long_term_memory=ltm,
       long_term_memory_mode="static_control"
   )
   # 每次 reply() 开始时自动检索，结束时自动记录
   ```

3. **both**: 两者结合
   ```python
   agent = ReActAgent(
       long_term_memory=ltm,
       long_term_memory_mode="both"
   )
   # 自动检索和记录，Agent 也可以主动调用
   ```

#### 2.2.5 Pipeline (工作流)

**定义**: Pipeline 定义了多个 Agent 的协作方式和执行顺序。

**Pipeline 模式**:

1. **顺序执行 (Sequential)**
   ```python
   result = await sequential_pipeline([
       agent1, agent2, agent3
   ], initial_msg)

   # 执行流程: agent1 → agent2 → agent3
   ```

2. **并行执行 (Fanout)**
   ```python
   results = await fanout_pipeline([
       agent1, agent2, agent3
   ], query_msg)

   # 执行流程: agent1 ║ agent2 ║ agent3 (并行)
   ```

3. **消息中心 (MsgHub)**
   ```python
   async with MsgHub([agent1, agent2, agent3]) as hub:
       await agent1(msg)  # agent1 的输出自动广播给 agent2 和 agent3
   ```

**复杂工作流示例**:

```python
async def complex_workflow(query):
    """复杂的分析工作流"""

    # 阶段 1: 数据预处理
    preprocessor = PreprocessorAgent(...)
    preprocessed = await preprocessor(query)

    # 阶段 2: 并行分析
    analyzers = [
        SentimentAgent(...),
        TopicAgent(...),
        EntityAgent(...)
    ]
    analyses = await fanout_pipeline(analyzers, preprocessed)

    # 阶段 3: 专家讨论
    experts = [ExpertA(...), ExpertB(...), ExpertC(...)]
    async with MsgHub(experts, announcement=analyses) as hub:
        await sequential_pipeline(experts)

    # 阶段 4: 最终总结
    summarizer = SummarizerAgent(...)
    final_result = await summarizer()

    return final_result
```

---

### 2.3 设计哲学

#### 2.3.1 显式优于隐式 (Explicit over Implicit)

**理念**: 开发者应该明确知道系统做了什么，而不是依赖"魔法"。

**示例对比**:

```python
# ❌ 隐式 (不推荐)
agent = Agent("gpt-4")  # 谁知道内部发生了什么?
response = agent("hello")  # 这是同步还是异步? 有没有调用工具?

# ✅ 显式 (AgentScope 方式)
agent = ReActAgent(
    name="assistant",
    model=OpenAIChatModel(model_name="gpt-4"),
    memory=InMemoryMemory(),
    formatter=OpenAIChatFormatter(),
    toolkit=Toolkit()
)
response = await agent(Msg("user", "hello", "user"))
# 清楚地知道: 这是异步的, 使用 OpenAI, 有记忆, 可以调用工具
```

**好处**:
- **易于理解**: 代码即文档
- **易于调试**: 知道在哪里查找问题
- **易于测试**: 可以 mock 任何组件

#### 2.3.2 组合优于继承 (Composition over Inheritance)

**理念**: 通过组合现有组件构建功能，而不是复杂的继承层次。

**示例**:

```python
# 通过组合不同组件创建不同类型的 Agent

# 基础对话 Agent
basic_agent = ReActAgent(
    model=model,
    memory=InMemoryMemory(),
    formatter=formatter,
    toolkit=Toolkit()
)

# 带长期记忆的 Agent
ltm_agent = ReActAgent(
    model=model,
    memory=InMemoryMemory(),
    formatter=formatter,
    toolkit=Toolkit(),
    long_term_memory=Mem0LTM(...)  # 添加长期记忆组件
)

# 带 RAG 的 Agent
rag_agent = ReActAgent(
    model=model,
    memory=InMemoryMemory(),
    formatter=formatter,
    toolkit=Toolkit(),
    knowledge=[kb1, kb2]  # 添加知识库组件
)

# 全功能 Agent
full_agent = ReActAgent(
    model=model,
    memory=InMemoryMemory(),
    formatter=formatter,
    toolkit=Toolkit(),
    long_term_memory=Mem0LTM(...),
    knowledge=[kb1, kb2],
    plan_notebook=PlanNotebook()
)
```

#### 2.3.3 约定优于配置 (Convention over Configuration)

**理念**: 提供合理的默认值，减少必需的配置。

**示例**:

```python
# 最简配置
agent = ReActAgent(
    name="assistant",
    sys_prompt="You are a helpful assistant",
    model=model,
    formatter=formatter
)
# 自动创建: InMemoryMemory(), Toolkit()

# 需要时才配置
agent = ReActAgent(
    name="assistant",
    sys_prompt="You are a helpful assistant",
    model=model,
    formatter=formatter,
    memory=CustomMemory(),  # 自定义记忆
    toolkit=custom_toolkit,  # 自定义工具集
    max_iters=20  # 覆盖默认值 10
)
```

**默认值设计原则**:
- **安全**: 默认值应该是安全的
- **常用**: 默认值应该适合大多数场景
- **可覆盖**: 所有默认值都可以被覆盖

#### 2.3.4 失败安全 (Fail Safe)

**理念**: 错误应该尽早暴露，而不是默默失败。

**错误处理策略**:

```python
# 1. 类型检查
async def reply(self, msg: Msg) -> Msg:
    if not isinstance(msg, Msg):
        raise TypeError(f"Expected Msg, got {type(msg)}")
    # ...

# 2. 参数验证
def register_tool_function(self, tool_func):
    if not callable(tool_func):
        raise ValueError("tool_func must be callable")
    if not hasattr(tool_func, "__name__"):
        raise ValueError("tool_func must have a name")
    # ...

# 3. 异常传播
async def call_tool_function(self, tool_call):
    try:
        result = await self.tools[tool_name](**kwargs)
    except Exception as e:
        # 不捕获异常，让它向上传播
        logger.error(f"Tool execution failed: {e}")
        raise
```

#### 2.3.5 开放封闭原则 (Open-Closed Principle)

**理念**: 对扩展开放，对修改封闭。

**实现方式**:

1. **抽象基类**: 定义接口，不限制实现
   ```python
   class ChatModelBase(ABC):
       @abstractmethod
       async def __call__(self, messages, **kwargs):
           """子类可以实现任何 LLM"""
   ```

2. **钩子系统**: 无需修改代码即可添加功能
   ```python
   # 添加自定义逻辑
   def my_logging_hook(self, kwargs):
       logger.info(f"Agent {self.name} called")
       return kwargs

   ReActAgent.register_class_hook("pre_reply", "my_log", my_logging_hook)
   ```

3. **插件式工具**: 动态注册工具
   ```python
   # 无需修改 Toolkit 代码
   toolkit.register_tool_function(my_new_tool)
   ```

---

### 2.4 异步优先架构

#### 2.4.1 为什么选择异步?

**传统同步模式的问题**:

```python
# 同步模式 - 阻塞执行
def agent_call(msg):
    # 1. 调用 LLM (耗时 2 秒)
    response = model.generate(msg)  # 阻塞 2 秒

    # 2. 调用工具 (耗时 3 秒)
    result = tool.execute()  # 阻塞 3 秒

    # 3. 再次调用 LLM (耗时 2 秒)
    final = model.generate(result)  # 阻塞 2 秒

    return final  # 总耗时: 2+3+2 = 7 秒

# 并发请求会相互阻塞
responses = [agent_call(msg1), agent_call(msg2), agent_call(msg3)]
# 总耗时: 7*3 = 21 秒
```

**异步模式的优势**:

```python
# 异步模式 - 非阻塞执行
async def agent_call(msg):
    # 1. 异步调用 LLM
    response = await model.generate(msg)  # 非阻塞

    # 2. 异步调用工具
    result = await tool.execute()  # 非阻塞

    # 3. 再次异步调用 LLM
    final = await model.generate(result)  # 非阻塞

    return final

# 并发请求可以交错执行
responses = await asyncio.gather(
    agent_call(msg1),
    agent_call(msg2),
    agent_call(msg3)
)
# 总耗时: ~7 秒 (并发执行)
```

#### 2.4.2 异步架构的核心组件

**1. 异步函数 (Async Functions)**

所有 I/O 操作都是异步的：

```python
class ReActAgent:
    # 核心方法都是异步
    async def observe(self, msg): ...
    async def reply(self, msg): ...
    async def print(self, msg): ...

    # 内部方法也是异步
    async def _reasoning(self): ...
    async def _acting(self, tool_calls): ...
```

**2. 异步迭代器 (Async Generators)**

支持流式输出：

```python
# 模型返回异步生成器
async def model_call(messages):
    async for chunk in api.stream(messages):
        yield ChatResponse(content=chunk, stream=True, is_last=False)
    yield ChatResponse(content=final, stream=True, is_last=True)

# Agent 处理流式响应
async for response in model_call(messages):
    await self.print(response, last=response.is_last)
```

**3. 并发执行 (Concurrent Execution)**

并行执行多个任务：

```python
# 并行执行多个工具
results = await asyncio.gather(*[
    toolkit.call_tool_function(tc1),
    toolkit.call_tool_function(tc2),
    toolkit.call_tool_function(tc3)
])

# 并行执行多个 Agent
responses = await asyncio.gather(*[
    agent1(msg),
    agent2(msg),
    agent3(msg)
])
```

#### 2.4.3 异步模式最佳实践

**1. 始终使用 `await`**

```python
# ❌ 错误: 忘记 await
response = agent(msg)  # 返回 coroutine 对象，而不是结果

# ✅ 正确
response = await agent(msg)
```

**2. 在异步函数中调用异步函数**

```python
# ❌ 错误: 在同步函数中调用异步函数
def process():
    result = await agent(msg)  # SyntaxError

# ✅ 正确
async def process():
    result = await agent(msg)
```

**3. 使用 `asyncio.gather` 并发执行**

```python
# ❌ 低效: 顺序执行
result1 = await slow_operation1()
result2 = await slow_operation2()
result3 = await slow_operation3()

# ✅ 高效: 并发执行
results = await asyncio.gather(
    slow_operation1(),
    slow_operation2(),
    slow_operation3()
)
```

**4. 错误处理**

```python
# 处理单个异步调用的错误
try:
    result = await agent(msg)
except Exception as e:
    logger.error(f"Agent failed: {e}")

# 处理并发调用的错误
results = await asyncio.gather(
    agent1(msg),
    agent2(msg),
    agent3(msg),
    return_exceptions=True  # 返回异常而不是抛出
)

for i, result in enumerate(results):
    if isinstance(result, Exception):
        logger.error(f"Agent {i} failed: {result}")
```

#### 2.4.4 性能优化技巧

**1. 避免不必要的 await**

```python
# ❌ 低效
async def process():
    data = await load_data()
    await asyncio.sleep(0)  # 不必要的 await
    result = await process_data(data)
    return result

# ✅ 高效
async def process():
    data = await load_data()
    result = await process_data(data)  # 直接处理
    return result
```

**2. 使用任务并发**

```python
# 创建任务而不是立即 await
task1 = asyncio.create_task(agent1(msg))
task2 = asyncio.create_task(agent2(msg))

# 做其他工作
do_something_else()

# 等待结果
result1 = await task1
result2 = await task2
```

**3. 流式处理大数据**

```python
# ❌ 一次性加载所有数据
async def process_all():
    data = await load_all_data()  # 可能很大
    for item in data:
        await process(item)

# ✅ 流式处理
async def process_stream():
    async for item in load_data_stream():
        await process(item)  # 逐个处理
```

---

## 第三章: Agent 模块深度解析

### 3.1 AgentBase 基类

**文件位置**: `src/agentscope/agent/_agent_base.py`

#### 3.1.1 类设计概览

AgentBase 是所有 Agent 的基类，定义了 Agent 的核心接口和行为。

**继承关系**:
```python
AgentBase(StateModule, metaclass=_AgentMeta)
```

**设计理念**:
1. **最小接口**: 只定义必须的抽象方法
2. **钩子扩展**: 通过钩子系统实现灵活扩展
3. **状态管理**: 继承 StateModule 支持序列化
4. **元类魔法**: 使用元类自动处理钩子继承

**核心属性**:

```python
class AgentBase(StateModule, metaclass=_AgentMeta):
    """Agent 基类

    核心属性:
        id: Agent 唯一标识 (使用 shortuuid 生成)
        _subscribers: 订阅者字典 {hub_id: [subscribers]}
        _reply_task: 当前正在执行的 reply 任务
        _reply_id: 当前 reply 的标识
        _stream_prefix: 流式输出的前缀缓存
        _disable_console_output: 是否禁用控制台输出
        msg_queue: 消息队列 (用于导出消息流)
    """

    # 类级别的钩子存储
    _class_pre_reply_hooks: OrderedDict = OrderedDict()
    _class_post_reply_hooks: OrderedDict = OrderedDict()
    _class_pre_print_hooks: OrderedDict = OrderedDict()
    _class_post_print_hooks: OrderedDict = OrderedDict()
    _class_pre_observe_hooks: OrderedDict = OrderedDict()
    _class_post_observe_hooks: OrderedDict = OrderedDict()

    def __init__(self) -> None:
        super().__init__()
        self.id = shortuuid.uuid()

        # 实例级别的钩子存储
        self._instance_pre_reply_hooks = OrderedDict()
        self._instance_post_reply_hooks = OrderedDict()
        # ... 其他钩子

        # 订阅者管理
        self._subscribers: dict[str, list[AgentBase]] = {}

        # 流式输出缓存
        self._stream_prefix = {}
```

#### 3.1.2 核心方法详解

**1. observe() - 观察方法**

**功能**: 接收消息但不生成回复，用于让 Agent "听到"其他 Agent 的消息。

**设计理念**:
- **被动感知**: Agent 被动接收信息
- **无副作用**: 只记录，不回复
- **钩子支持**: 支持 pre/post 钩子

**实现**:
```python
async def observe(self, msg: Msg | list[Msg] | None) -> None:
    """观察消息

    执行流程:
    1. 执行 pre_observe 钩子
    2. 将消息添加到记忆 (子类实现)
    3. 执行 post_observe 钩子

    Args:
        msg: 要观察的消息
    """
    if msg is None:
        return

    # 准备参数
    kwargs = {"msg": msg}

    # 执行 pre_observe 钩子
    kwargs = await self._execute_hooks("pre_observe", kwargs)

    # 调用子类实现
    await self._observe_impl(kwargs["msg"])

    # 执行 post_observe 钩子
    await self._execute_hooks("post_observe", {"msg": kwargs["msg"]})
```

**使用场景**:
```python
# 场景 1: MsgHub 自动调用
async with MsgHub([agent1, agent2, agent3]) as hub:
    await agent1(msg)
    # agent2 和 agent3 自动调用 observe(agent1_response)

# 场景 2: 手动让 Agent 观察
expert_msg = await expert(query)
await observer.observe(expert_msg)  # observer 记录但不回复
```

**2. reply() - 回复方法**

**功能**: Agent 的核心逻辑，生成对输入的回复。

**设计理念**:
- **抽象方法**: 必须由子类实现
- **异步执行**: 支持并发
- **灵活输入**: 接受任意参数

**签名**:
```python
@abstractmethod
async def reply(self, *args: Any, **kwargs: Any) -> Msg:
    """生成回复 (由子类实现)

    Args:
        *args: 位置参数
        **kwargs: 关键字参数

    Returns:
        Msg: 回复消息
    """
    raise NotImplementedError
```

**子类实现示例**:
```python
# ReActAgent 的实现
async def reply(
    self,
    msg: Msg | list[Msg] | None = None,
    structured_model: Type[BaseModel] | None = None
) -> Msg:
    """ReAct 循环实现"""

    # 1. 添加到记忆
    await self.memory.add(msg)

    # 2. 长期记忆检索 (如果启用)
    if self._static_control:
        retrieved = await self.long_term_memory.retrieve(...)
        await self.memory.add(Msg("system", retrieved, "system"))

    # 3. 知识库检索 (如果启用)
    for kb in self.knowledge:
        docs = await kb.retrieve(...)

    # 4. ReAct 循环
    for i in range(self.max_iters):
        # 推理
        msg_reasoning = await self._reasoning(structured_model)

        # 提取工具调用
        tool_calls = msg_reasoning.get_content_blocks("tool_use")

        if tool_calls:
            # 执行工具
            await self._acting(tool_calls)
        else:
            # 没有工具调用，结束循环
            reply_msg = msg_reasoning
            break

    # 5. 记录到长期记忆 (如果启用)
    if self._static_control:
        await self.long_term_memory.record(...)

    return reply_msg
```

**3. __call__() - 调用方法**

**功能**: Agent 的完整执行流程，包括钩子、广播、打印等。

**设计理念**:
- **完整流程**: 将所有步骤组合在一起
- **钩子集成**: 在关键点执行钩子
- **消息传播**: 自动广播给订阅者

**实现**:
```python
async def __call__(self, *args: Any, **kwargs: Any) -> Msg:
    """完整的 Agent 执行流程

    执行步骤:
    1. 执行 pre_reply 钩子
    2. 调用 reply() 生成回复
    3. 执行 post_reply 钩子
    4. 广播给订阅者
    5. 打印消息
    6. 返回结果
    """
    # 1. 准备参数
    call_kwargs = kwargs.copy()

    # 2. 执行 pre_reply 钩子
    call_kwargs = await self._execute_hooks("pre_reply", call_kwargs)

    # 3. 调用 reply()
    response = await self.reply(*args, **call_kwargs)

    # 4. 执行 post_reply 钩子
    response = await self._execute_hooks(
        "post_reply",
        {"kwargs": call_kwargs, "output": response}
    )

    # 5. 广播给订阅者
    await self._broadcast_to_subscribers(response)

    # 6. 打印消息
    await self.print(response)

    return response
```

**使用方式**:
```python
# 方式 1: 直接调用
response = await agent(msg)

# 方式 2: 作为函数使用
agent_func = agent
response = await agent_func(msg)

# 方式 3: 在 Pipeline 中使用
result = await sequential_pipeline([agent1, agent2, agent3], msg)
```

**4. print() - 打印方法**

**功能**: 显示消息，支持流式输出。

**设计理念**:
- **可配置**: 可以禁用输出
- **流式支持**: 支持流式消息的增量显示
- **钩子扩展**: 可以通过钩子修改输出行为

**实现**:
```python
async def print(self, msg: Msg, last: bool = True) -> None:
    """打印消息

    Args:
        msg: 要打印的消息
        last: 是否是流式消息的最后一块
    """
    if self._disable_console_output:
        return

    kwargs = {"msg": msg, "last": last}

    # 执行 pre_print 钩子
    kwargs = await self._execute_hooks("pre_print", kwargs)

    # 实际打印逻辑
    await self._print_impl(kwargs["msg"], kwargs["last"])

    # 执行 post_print 钩子
    await self._execute_hooks("post_print", kwargs)

async def _print_impl(self, msg: Msg, last: bool) -> None:
    """实际打印逻辑"""

    # 处理流式输出
    if hasattr(msg, "stream") and msg.stream and not last:
        # 流式输出：累积显示
        if msg.id not in self._stream_prefix:
            self._stream_prefix[msg.id] = {"text": "", "audio": None}

        # 提取文本内容
        text = msg.get_text_content()
        if text:
            # 只打印新增的部分
            new_text = text[len(self._stream_prefix[msg.id]["text"]):]
            print(new_text, end="", flush=True)
            self._stream_prefix[msg.id]["text"] = text
    else:
        # 非流式或最后一块：完整打印
        print(f"\n{msg.name}: {msg.get_text_content()}")

        # 清理流式缓存
        if msg.id in self._stream_prefix:
            del self._stream_prefix[msg.id]
```

**流式输出示例**:
```python
# Agent 生成流式响应
async for chunk in agent.reply_stream(msg):
    await agent.print(chunk, last=chunk.is_last)

# 输出效果:
# Agent: Hello,      (第一块)
# Agent: Hello, how  (第二块，只打印 " how")
# Agent: Hello, how are you? (最后一块，只打印 " are you?")
```

#### 3.1.3 订阅者机制

**设计目的**: 实现 Agent 间的自动消息传播。

**核心方法**:

```python
def set_subscribers(
    self,
    hub_id: str,
    subscribers: list[AgentBase]
) -> None:
    """设置订阅者

    Args:
        hub_id: MsgHub 的标识
        subscribers: 订阅者列表
    """
    self._subscribers[hub_id] = subscribers

def remove_subscribers(self, hub_id: str) -> None:
    """移除订阅者组"""
    if hub_id in self._subscribers:
        del self._subscribers[hub_id]

async def _broadcast_to_subscribers(self, msg: Msg) -> None:
    """广播消息给所有订阅者"""
    for subscribers in self._subscribers.values():
        await asyncio.gather(*[
            subscriber.observe(msg)
            for subscriber in subscribers
        ])
```

**工作原理**:

```python
# MsgHub 设置订阅关系
async with MsgHub([agent1, agent2, agent3]) as hub:
    # hub 内部调用:
    # agent1.set_subscribers(hub.id, [agent2, agent3])
    # agent2.set_subscribers(hub.id, [agent1, agent3])
    # agent3.set_subscribers(hub.id, [agent1, agent2])

    # 当 agent1 回复时
    response = await agent1(msg)
    # agent1.__call__() 内部自动调用:
    # await agent1._broadcast_to_subscribers(response)
    # 这会自动让 agent2 和 agent3 观察到这条消息
```

#### 3.1.4 状态管理

AgentBase 继承自 StateModule，支持完整的状态序列化。

**可序列化的状态**:
```python
def state_dict(self) -> dict:
    """序列化 Agent 状态"""
    return {
        "id": self.id,
        "class_name": self.__class__.__name__,
        "subscribers": {
            hub_id: [agent.id for agent in agents]
            for hub_id, agents in self._subscribers.items()
        },
        # 子类添加更多状态
    }

def load_state_dict(self, state_dict: dict) -> None:
    """加载 Agent 状态"""
    self.id = state_dict["id"]
    # 订阅者需要在加载后重新连接
    # 子类加载更多状态
```

**使用示例**:
```python
# 保存 Agent 状态
state = agent.state_dict()
with open("agent_state.json", "w") as f:
    json.dump(state, f)

# 加载 Agent 状态
with open("agent_state.json", "r") as f:
    state = json.load(f)

new_agent = ReActAgent(...)  # 创建相同类型的 Agent
new_agent.load_state_dict(state)
```

---

### 3.2 钩子系统详解

#### 3.2.1 钩子系统设计理念

**什么是钩子?**
钩子(Hook)是在特定执行点插入自定义逻辑的机制，类似于事件监听器。

**为什么需要钩子?**
1. **无侵入扩展**: 不修改源代码即可添加功能
2. **关注点分离**: 将核心逻辑和辅助功能分离
3. **灵活配置**: 可以动态添加/移除钩子
4. **可组合**: 多个钩子可以链式执行

**设计模式**: 观察者模式 + 责任链模式

#### 3.2.2 支持的钩子类型

AgentBase 支持 6 种钩子：

```python
supported_hook_types = [
    "pre_reply",     # reply() 执行前
    "post_reply",    # reply() 执行后
    "pre_print",     # print() 执行前
    "post_print",    # print() 执行后
    "pre_observe",   # observe() 执行前
    "post_observe",  # observe() 执行后
]
```

ReActAgentBase 额外支持：

```python
supported_hook_types = [
    *AgentBase.supported_hook_types,
    "pre_reasoning",   # _reasoning() 执行前
    "post_reasoning",  # _reasoning() 执行后
    "pre_acting",      # _acting() 执行前
    "post_acting",     # _acting() 执行后
]
```

#### 3.2.3 钩子函数签名

**pre_* 钩子**:
```python
def pre_hook(
    self: AgentBase,           # Agent 实例
    kwargs: dict[str, Any]     # 方法的参数字典
) -> dict[str, Any] | None:    # 返回修改后的参数或 None
    """前置钩子

    功能:
    - 检查/修改参数
    - 记录日志
    - 执行预处理

    返回:
    - dict: 修改后的参数，传递给下一个钩子或原方法
    - None: 不修改参数
    """
    # 示例：记录调用
    logger.info(f"Agent {self.name} called with {kwargs}")

    # 示例：修改参数
    if "msg" in kwargs:
        kwargs["msg"] = preprocess(kwargs["msg"])

    return kwargs
```

**post_* 钩子**:
```python
def post_hook(
    self: AgentBase,           # Agent 实例
    kwargs: dict[str, Any],    # 原始参数
    output: Any                # 方法的输出
) -> Any | None:               # 返回修改后的输出或 None
    """后置钩子

    功能:
    - 检查/修改输出
    - 记录结果
    - 执行后处理

    返回:
    - Any: 修改后的输出，传递给下一个钩子或调用者
    - None: 不修改输出
    """
    # 示例：记录结果
    logger.info(f"Agent {self.name} returned {output}")

    # 示例：修改输出
    if isinstance(output, Msg):
        output = postprocess(output)

    return output
```

#### 3.2.4 注册钩子

**类级钩子** (影响该类的所有实例):

```python
# 方法 1: 使用类方法
ReActAgent.register_class_hook(
    hook_type="pre_reply",
    hook_name="my_logging_hook",
    hook_func=my_logging_hook
)

# 方法 2: 直接操作字典 (不推荐)
ReActAgent._class_pre_reply_hooks["my_hook"] = my_hook_func
```

**实例级钩子** (只影响当前实例):

```python
# 创建 Agent 实例
agent = ReActAgent(...)

# 注册实例级钩子
agent.register_instance_hook(
    hook_type="post_reply",
    hook_name="my_hook",
    hook_func=my_hook_func
)
```

**钩子执行顺序**:
```
类级钩子 (按注册顺序) → 实例级钩子 (按注册顺序)
```

#### 3.2.5 钩子实战示例

**示例 1: 日志记录钩子**

```python
def logging_pre_reply_hook(self, kwargs):
    """记录每次 Agent 调用"""
    logger.info(f"[{datetime.now()}] Agent {self.name} started")
    logger.debug(f"Input: {kwargs}")
    return kwargs

def logging_post_reply_hook(self, kwargs, output):
    """记录每次 Agent 响应"""
    logger.info(f"[{datetime.now()}] Agent {self.name} finished")
    logger.debug(f"Output: {output}")
    return output

# 全局注册
ReActAgent.register_class_hook("pre_reply", "logging_pre", logging_pre_reply_hook)
ReActAgent.register_class_hook("post_reply", "logging_post", logging_post_reply_hook)
```

**示例 2: 性能监控钩子**

```python
import time

def performance_pre_hook(self, kwargs):
    """记录开始时间"""
    kwargs["_start_time"] = time.time()
    return kwargs

def performance_post_hook(self, kwargs, output):
    """计算执行时间"""
    if "_start_time" in kwargs:
        elapsed = time.time() - kwargs["_start_time"]
        logger.info(f"Agent {self.name} took {elapsed:.2f}s")
    return output

ReActAgent.register_class_hook("pre_reply", "perf_start", performance_pre_hook)
ReActAgent.register_class_hook("post_reply", "perf_end", performance_post_hook)
```

**示例 3: 消息过滤钩子**

```python
def content_filter_hook(self, kwargs):
    """过滤敏感内容"""
    if "msg" in kwargs:
        msg = kwargs["msg"]
        text = msg.get_text_content()

        # 检查敏感词
        if contains_sensitive_words(text):
            logger.warning(f"Blocked sensitive content from {msg.name}")
            # 修改消息
            kwargs["msg"] = Msg(
                msg.name,
                "[此消息包含敏感内容已被过滤]",
                msg.role
            )

    return kwargs

# 为特定 Agent 注册
important_agent.register_instance_hook(
    "pre_reply",
    "content_filter",
    content_filter_hook
)
```

**示例 4: 工具调用审计钩子**

```python
def tool_audit_pre_hook(self, kwargs):
    """审计工具调用"""
    tool_calls = kwargs.get("tool_calls", [])

    for tc in tool_calls:
        logger.info(f"Agent {self.name} calling tool: {tc['name']}")

        # 检查危险操作
        if tc["name"] in ["execute_shell_command", "write_file"]:
            # 需要人工确认
            confirmation = input(f"Allow {tc['name']}? (y/n): ")
            if confirmation.lower() != "y":
                raise PermissionError(f"Tool {tc['name']} blocked by user")

    return kwargs

ReActAgent.register_class_hook("pre_acting", "tool_audit", tool_audit_pre_hook)
```

**示例 5: 自动重试钩子**

```python
def auto_retry_hook(self, kwargs, output):
    """自动重试失败的调用"""
    # 检查是否失败
    if isinstance(output, Msg):
        text = output.get_text_content()
        if "error" in text.lower() or "failed" in text.lower():
            logger.warning(f"Agent {self.name} failed, retrying...")

            # 重试 (注意：这里简化了，实际需要更复杂的逻辑)
            # return self.reply(**kwargs)  # 递归可能导致问题

            # 更好的做法是在外层捕获并重试
            output.metadata = output.metadata or {}
            output.metadata["needs_retry"] = True

    return output

# 注册后置钩子
agent.register_instance_hook("post_reply", "auto_retry", auto_retry_hook)
```

#### 3.2.6 钩子的高级用法

**条件钩子**:

```python
def conditional_hook(self, kwargs):
    """只在特定条件下执行"""
    # 只在包含特定关键词时记录
    if "msg" in kwargs:
        text = kwargs["msg"].get_text_content()
        if "important" in text.lower():
            logger.critical(f"Important message detected: {text}")

    return kwargs
```

**钩子组合**:

```python
# 定义多个钩子实现不同功能
hooks = [
    ("logging", logging_hook),
    ("performance", performance_hook),
    ("audit", audit_hook),
    ("retry", retry_hook),
]

# 批量注册
for name, func in hooks:
    ReActAgent.register_class_hook("pre_reply", name, func)
```

**移除钩子**:

```python
# 移除类级钩子
if "my_hook" in ReActAgent._class_pre_reply_hooks:
    del ReActAgent._class_pre_reply_hooks["my_hook"]

# 移除实例级钩子
if "my_hook" in agent._instance_pre_reply_hooks:
    del agent._instance_pre_reply_hooks["my_hook"]
```

---

### 3.3 ReActAgent 实现

**文件位置**: `src/agentscope/agent/_react_agent.py`

#### 3.3.1 ReAct 模式介绍

**什么是 ReAct?**

ReAct = **Rea**soning + **Act**ing

这是一种让 LLM 交替进行推理(Reasoning)和行动(Acting)的范式，特别适合需要使用工具的复杂任务。

**传统模式 vs ReAct 模式**:

```
传统模式:
用户输入 → LLM 直接回答 → 结束
问题: LLM 不能访问实时信息、执行代码、操作文件等

ReAct 模式:
用户输入 → LLM 推理(思考下一步) → 调用工具 → 观察结果 →
         LLM 推理(基于结果思考) → 调用工具 → 观察结果 →
         ... (循环) ...
         LLM 推理(给出最终答案) → 结束
优势: LLM 可以使用工具扩展能力边界
```

**ReAct 示例**:

```
用户: 今天北京的天气怎么样?

循环 1:
  推理: 我需要查询天气信息
  行动: 调用 get_weather(city="北京")
  观察: {"temperature": 15, "condition": "晴"}

循环 2:
  推理: 已获得天气信息，可以回答用户
  行动: 调用 generate_response(response="今天北京天气晴朗，温度15度")
  观察: (结束)

输出: 今天北京天气晴朗，温度15度
```

#### 3.3.2 ReActAgent 初始化

**完整的初始化参数**:

```python
def __init__(
    self,
    name: str,                          # Agent 名称
    sys_prompt: str,                    # 系统提示
    model: ChatModelBase,               # LLM 模型
    formatter: FormatterBase,           # 消息格式化器
    toolkit: Toolkit | None = None,    # 工具包
    memory: MemoryBase | None = None,  # 短期记忆
    long_term_memory: LongTermMemoryBase | None = None,  # 长期记忆
    long_term_memory_mode: Literal[
        "agent_control",      # Agent 自主控制
        "static_control",     # 自动控制
        "both",              # 两者结合
    ] = "both",
    enable_meta_tool: bool = False,    # 是否启用元工具
    parallel_tool_calls: bool = False, # 是否并行执行工具
    knowledge: KnowledgeBase | list[KnowledgeBase] | None = None,  # 知识库
    enable_rewrite_query: bool = True,  # 是否重写查询
    plan_notebook: PlanNotebook | None = None,  # 计划笔记本
    print_hint_msg: bool = False,       # 是否打印提示消息
    max_iters: int = 10,                # 最大循环次数
) -> None:
    """初始化 ReActAgent

    设计要点:
    1. 所有复杂功能都是可选的 (知识库、计划、长期记忆等)
    2. 使用依赖注入 (传入 model、formatter 等)
    3. 提供合理的默认值 (max_iters=10)
    """
```

**参数详解**:

1. **name** - Agent 名称
   - 用途: 标识 Agent、日志、消息发送者
   - 示例: `"assistant"`, `"researcher"`, `"code_expert"`

2. **sys_prompt** - 系统提示
   - 用途: 定义 Agent 的角色和行为规范
   - 示例:
     ```python
     sys_prompt = """你是一个专业的 Python 编程助手。

     你的职责:
     1. 帮助用户编写、调试 Python 代码
     2. 解释代码的工作原理
     3. 提供最佳实践建议

     你的特点:
     - 专业: 提供准确的技术信息
     - 友好: 使用通俗易懂的语言
     - 高效: 快速定位问题
     """
     ```

3. **model** - LLM 模型
   ```python
   # OpenAI
   model = OpenAIChatModel(
       model_name="gpt-4",
       api_key="your_key"
   )

   # Anthropic Claude
   model = AnthropicChatModel(
       model_name="claude-3-opus-20240229",
       api_key="your_key"
   )

   # 阿里云通义千问
   model = DashScopeChatModel(
       model_name="qwen-max",
       api_key="your_key"
   )
   ```

4. **formatter** - 消息格式化器
   ```python
   # 必须与 model 匹配
   formatter = OpenAIChatFormatter()  # 配合 OpenAI
   formatter = AnthropicChatFormatter()  # 配合 Anthropic
   formatter = DashScopeChatFormatter()  # 配合 DashScope
   ```

5. **toolkit** - 工具包
   ```python
   toolkit = Toolkit()

   # 注册自定义工具
   toolkit.register_tool_function(my_tool)

   # 注册 MCP 工具
   await toolkit.register_mcp_client(mcp_client)
   ```

6. **memory** - 短期记忆
   ```python
   # 默认: InMemoryMemory
   memory = InMemoryMemory()

   # 自定义记忆
   memory = RedisMemory(redis_url="...")
   ```

7. **long_term_memory** - 长期记忆
   ```python
   ltm = Mem0LongTermMemory(
       api_key="your_mem0_key",
       org_id="your_org",
       project_id="your_project"
   )
   ```

8. **long_term_memory_mode** - 长期记忆模式
   - `"agent_control"`: Agent 通过工具主动调用
   - `"static_control"`: 每次 reply 自动检索和记录
   - `"both"`: 两者结合

9. **knowledge** - 知识库
   ```python
   # 单个知识库
   kb = SimpleKnowledgeBase(...)
   knowledge = kb

   # 多个知识库
   knowledge = [kb_python, kb_javascript, kb_general]
   ```

10. **max_iters** - 最大循环次数
    - 防止无限循环
    - 简单任务: 5-10
    - 复杂任务: 10-20

#### 3.3.3 ReAct 循环核心实现

**reply() 方法全流程**:

```python
async def reply(
    self,
    msg: Msg | list[Msg] | None = None,
    structured_model: Type[BaseModel] | None = None
) -> Msg:
    """ReActAgent 的核心逻辑

    执行流程:
    1. 预处理: 添加消息到记忆
    2. 增强: 长期记忆检索、知识库检索、计划提示
    3. ReAct 循环: 推理 → 执行 → 观察 → 推理 → ...
    4. 后处理: 记录到长期记忆
    """

    # ========== 1. 预处理 ==========

    # 将输入添加到记忆
    if msg is not None:
        await self.memory.add(msg)

    # 提取查询文本 (用于检索)
    query_text = ""
    if msg:
        if isinstance(msg, list):
            query_text = msg[-1].get_text_content() if msg else ""
        else:
            query_text = msg.get_text_content() or ""

    # ========== 2. 增强 ==========

    # 2.1 长期记忆检索 (static_control 模式)
    if self._static_control and self.long_term_memory:
        retrieved = await self.long_term_memory.retrieve(
            query=query_text,
            agent_id=self.agent_id,
            limit=5
        )

        if retrieved:
            hint_msg = Msg(
                name="system",
                content=f"相关历史信息:\n{retrieved}",
                role="system"
            )
            await self.memory.add(hint_msg)

            if self.print_hint_msg:
                await self.print(hint_msg)

    # 2.2 知识库检索
    if self.knowledge:
        # 查询重写 (可选)
        if self.enable_rewrite_query:
            rewrite_prompt = f"""请将以下用户查询重写为更适合检索的形式:

            原始查询: {query_text}

            要求:
            - 提取关键词
            - 转换为陈述句
            - 保持语义不变
            """

            rewrite_response = await self.model([
                Msg("system", rewrite_prompt, "system")
            ], structured_model=_QueryRewriteModel)

            query_text = rewrite_response.rewritten_query

        # 从每个知识库检索
        all_docs = []
        for kb in (self.knowledge if isinstance(self.knowledge, list) else [self.knowledge]):
            docs = await kb.retrieve(
                query=query_text,
                limit=3,
                score_threshold=0.7
            )
            all_docs.extend(docs)

        # 添加到记忆
        if all_docs:
            docs_text = "\n\n".join([
                f"文档 {i+1} (相似度: {doc.score:.2f}):\n{doc.metadata.content['text']}"
                for i, doc in enumerate(all_docs)
            ])

            hint_msg = Msg(
                name="system",
                content=f"相关知识库信息:\n{docs_text}",
                role="system"
            )
            await self.memory.add(hint_msg)

            if self.print_hint_msg:
                await self.print(hint_msg)

    # ========== 3. ReAct 循环 ==========

    reply_msg = None

    for iteration in range(self.max_iters):
        logger.debug(f"ReAct iteration {iteration + 1}/{self.max_iters}")

        # 3.1 推理阶段
        msg_reasoning = await self._reasoning(structured_model)

        # 3.2 检查是否调用工具
        tool_calls = msg_reasoning.get_content_blocks("tool_use")

        if tool_calls:
            # 3.3 执行阶段
            tool_results = await self._acting(tool_calls)

            # 3.4 添加结果到记忆
            await self.memory.add(tool_results)
        else:
            # 没有工具调用，说明是最终回复
            reply_msg = msg_reasoning
            break
    else:
        # 达到最大循环次数
        logger.warning(f"Reached max iterations ({self.max_iters})")
        reply_msg = msg_reasoning

    # ========== 4. 后处理 ==========

    # 4.1 记录到长期记忆 (static_control 模式)
    if self._static_control and self.long_term_memory:
        messages_to_record = []
        if msg:
            messages_to_record.append(msg if not isinstance(msg, list) else msg[-1])
        messages_to_record.append(reply_msg)

        await self.long_term_memory.record(
            messages=messages_to_record,
            agent_id=self.agent_id
        )

    return reply_msg
```

**推理阶段 (_reasoning)**:

```python
async def _reasoning(
    self,
    structured_model: Type[BaseModel] | None = None
) -> Msg:
    """推理阶段: 调用 LLM 生成思考和行动

    执行流程:
    1. 获取计划提示 (如果有)
    2. 获取记忆
    3. 添加系统提示
    4. 格式化消息
    5. 调用模型
    6. 处理响应 (流式或非流式)
    """

    # 执行 pre_reasoning 钩子
    kwargs = {}
    kwargs = await self._execute_hooks("pre_reasoning", kwargs)

    # 1. 获取记忆
    memory = await self.memory.get_memory()

    # 2. 添加系统提示
    if self.sys_prompt:
        memory = [
            Msg("system", self.sys_prompt, "system"),
            *memory
        ]

    # 3. 获取计划提示
    if self.plan_notebook:
        plan_hint = await self.plan_notebook.get_current_hint()
        if plan_hint:
            memory.append(plan_hint)

            if self.print_hint_msg:
                await self.print(plan_hint)

    # 4. 格式化消息
    formatted_msgs = await self.formatter.format(
        msgs=memory,
        tools=self.toolkit.json_schemas
    )

    # 5. 调用模型
    model_response = await self.model(
        messages=formatted_msgs,
        tools=self.toolkit.json_schemas,
        structured_model=structured_model
    )

    # 6. 处理响应
    if isinstance(model_response, AsyncGenerator):
        # 流式响应
        msg_reasoning = None
        async for chunk in model_response:
            # 打印流式块
            await self.print(chunk, last=chunk.is_last)

            if chunk.is_last:
                msg_reasoning = chunk.to_msg(self.name)

        if msg_reasoning is None:
            raise RuntimeError("No response from streaming model")
    else:
        # 非流式响应
        msg_reasoning = model_response.to_msg(self.name)
        await self.print(msg_reasoning)

    # 添加到记忆
    await self.memory.add(msg_reasoning)

    # 执行 post_reasoning 钩子
    await self._execute_hooks("post_reasoning", {"output": msg_reasoning})

    return msg_reasoning
```

**执行阶段 (_acting)**:

```python
async def _acting(
    self,
    tool_calls: list[ToolUseBlock]
) -> list[Msg]:
    """执行阶段: 调用工具并获取结果

    执行流程:
    1. 执行 pre_acting 钩子
    2. 并行/顺序执行工具
    3. 收集结果
    4. 构造工具结果消息
    5. 执行 post_acting 钩子
    """

    # 执行 pre_acting 钩子
    kwargs = {"tool_calls": tool_calls}
    kwargs = await self._execute_hooks("pre_acting", kwargs)
    tool_calls = kwargs["tool_calls"]

    # 执行工具
    if self.parallel_tool_calls:
        # 并行执行
        tool_responses = await asyncio.gather(*[
            self._call_single_tool(tc)
            for tc in tool_calls
        ])
    else:
        # 顺序执行
        tool_responses = []
        for tc in tool_calls:
            response = await self._call_single_tool(tc)
            tool_responses.append(response)

    # 构造工具结果消息
    tool_result_msgs = []
    for tc, tr in zip(tool_calls, tool_responses):
        msg = Msg(
            name=self.name,
            content=[
                ToolResultBlock(
                    type="tool_result",
                    id=tc["id"],
                    output=tr.content
                )
            ],
            role="assistant"
        )
        tool_result_msgs.append(msg)

    # 执行 post_acting 钩子
    await self._execute_hooks("post_acting", {"tool_results": tool_result_msgs})

    return tool_result_msgs

async def _call_single_tool(self, tool_call: ToolUseBlock) -> ToolResponse:
    """调用单个工具 (支持流式)"""
    final_response = None

    async for response in self.toolkit.call_tool_function(tool_call):
        if response.stream:
            # 流式工具: 打印每一块
            await self.print(
                Msg(self.name, response.content, "assistant"),
                last=response.is_last
            )

        if response.is_last:
            final_response = response

    return final_response
```

#### 3.3.4 特殊功能

**1. 结构化输出**

```python
from pydantic import BaseModel, Field

class AnalysisResult(BaseModel):
    """分析结果的结构化模型"""
    sentiment: str = Field(description="情感倾向: positive/negative/neutral")
    topics: list[str] = Field(description="主要话题列表")
    confidence: float = Field(description="置信度 0-1")

# 使用结构化输出
response = await agent(
    Msg("user", "分析这篇文章", "user"),
    structured_model=AnalysisResult
)

# response 会强制符合 AnalysisResult 结构
```

**2. 完成函数 (Finish Function)**

ReActAgent 自动注册一个 `generate_response` 函数，用于结束循环:

```python
def generate_response(self, response: str) -> ToolResponse:
    """生成最终回复

    Args:
        response: 要返回给用户的回复内容

    返回:
        ToolResponse 包含回复文本
    """
    return ToolResponse(
        content=[TextBlock(type="text", text=response)]
    )
```

LLM 会这样调用:
```json
{
  "type": "tool_use",
  "name": "generate_response",
  "input": {
    "response": "今天北京天气晴朗，温度15度"
  }
}
```

**3. 元工具 (Meta Tool)**

如果 `enable_meta_tool=True`，会注册 `reset_equipped_tools` 函数:

```python
def reset_equipped_tools(self, group_names: list[str]) -> ToolResponse:
    """让 Agent 动态选择工具组

    Args:
        group_names: 要激活的工具组名称列表

    示例:
        Agent 在处理文件任务时:
        reset_equipped_tools(["file", "text_processing"])

        Agent 在处理网络任务时:
        reset_equipped_tools(["web", "api"])
    """
    return self.toolkit.reset_equipped_tools(group_names)
```

---

### 3.4 UserAgent 实现

**文件位置**: `src/agentscope/agent/_user_agent.py`

#### 3.4.1 UserAgent 设计

**用途**: 与人类用户交互，获取用户输入。

**设计理念**:
1. **输入源可配置**: 终端、Web UI、Studio 等
2. **无状态**: 不需要记忆、模型等
3. **简单直接**: 只做一件事—获取用户输入

#### 3.4.2 输入方法

**支持的输入源**:

1. **终端输入** (默认)
   ```python
   user = UserAgent(name="user")
   response = await user()
   # 提示用户在终端输入
   ```

2. **Studio 输入**
   ```python
   from agentscope.agent import StudioUserInput

   # 为所有 UserAgent 设置 Studio 输入
   UserAgent.override_class_input_method(
       StudioUserInput(studio_url="http://localhost:5000")
   )

   user = UserAgent(name="user")
   response = await user()
   # 从 Studio Web 界面获取输入
   ```

3. **自定义输入**
   ```python
   def custom_input_method(agent_name: str) -> str:
       """自定义输入方法"""
       # 可以从任何来源获取输入
       return get_input_from_custom_source()

   user = UserAgent(name="user")
   user.override_instance_input_method(custom_input_method)
   ```

#### 3.4.3 完整示例

```python
# 场景: 用户与 AI 助手对话

user_agent = UserAgent(name="user")
ai_agent = ReActAgent(
    name="assistant",
    sys_prompt="You are a helpful assistant",
    model=model,
    formatter=formatter
)

# 对话循环
while True:
    # 获取用户输入
    user_msg = await user_agent()

    # 检查退出
    if user_msg.get_text_content().lower() in ["quit", "exit", "bye"]:
        break

    # AI 回复
    ai_response = await ai_agent(user_msg)

    # 继续下一轮
```

---

## 第四章: Model 模块详细分析

**文件位置**: `src/agentscope/model/`

### 4.1 ChatModelBase 基类

**文件**: `_model_base.py`

#### 4.1.1 设计理念

ChatModelBase 是所有聊天模型的抽象基类，定义了统一的接口。

**核心设计原则**:
1. **模型无关**: 不绑定特定厂商的 API
2. **统一接口**: 所有模型使用相同的调用方式
3. **异步优先**: 支持异步调用
4. **流式支持**: 支持流式和非流式输出
5. **工具调用**: 原生支持工具调用（Function Calling）

#### 4.1.2 核心方法

```python
from abc import ABC, abstractmethod
from typing import AsyncGenerator

class ChatModelBase(ABC):
    """聊天模型基类"""

    def __init__(
        self,
        model_name: str,
        api_key: str | None = None,
        api_base: str | None = None,
        **kwargs
    ):
        """初始化模型

        Args:
            model_name: 模型名称
            api_key: API 密钥
            api_base: API 基础 URL
        """
        self.model_name = model_name
        self.api_key = api_key
        self.api_base = api_base

    @abstractmethod
    async def __call__(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        stream: bool = False,
        **kwargs
    ) -> ChatResponse | AsyncGenerator[ChatResponse, None]:
        """调用模型生成回复

        Args:
            messages: 消息列表（已格式化）
            tools: 工具列表（JSON Schema 格式）
            stream: 是否使用流式输出
            **kwargs: 其他模型参数（temperature、max_tokens 等）

        Returns:
            ChatResponse 或 AsyncGenerator[ChatResponse]
        """
        raise NotImplementedError
```

### 4.2 响应对象设计

**文件**: `_model_response.py`

#### 4.2.1 ChatResponse 类

```python
from dataclasses import dataclass
from agentscope.message import ContentBlock

@dataclass
class ChatResponse:
    """聊天模型响应对象

    属性:
        id: 响应 ID
        content: 内容块列表
        created_at: 创建时间
        type: 响应类型（"chat"）
        stream: 是否为流式响应
        is_last: 是否为流式响应的最后一块
        usage: Token 使用情况
        metadata: 元数据
    """
    id: str
    content: list[ContentBlock]
    created_at: str
    type: str = "chat"
    stream: bool = False
    is_last: bool = True
    usage: dict | None = None
    metadata: dict | None = None

    def to_msg(self, name: str) -> Msg:
        """转换为 Msg 对象

        Args:
            name: 消息发送者名称

        Returns:
            Msg 对象
        """
        return Msg(
            name=name,
            content=self.content,
            role="assistant",
            metadata={
                "response_id": self.id,
                "usage": self.usage,
                **(self.metadata or {})
            }
        )

    def get_text_content(self) -> str:
        """提取文本内容"""
        text_parts = []
        for block in self.content:
            if block["type"] == "text":
                text_parts.append(block["text"])
            elif block["type"] == "thinking":
                # 思考块也是文本
                text_parts.append(block["thinking"])
        return "".join(text_parts)
```

### 4.3 各厂商模型实现

#### 4.3.1 DashScope 模型（阿里云通义千问）

**文件**: `_dashscope_model.py`

```python
from dashscope import Generation
import dashscope

class DashScopeChatModel(ChatModelBase):
    """阿里云通义千问模型

    支持的模型:
    - qwen-max: 最强大的通义千问模型
    - qwen-plus: 平衡性能和成本
    - qwen-turbo: 快速响应
    """

    def __init__(
        self,
        model_name: str = "qwen-max",
        api_key: str | None = None,
        stream: bool = False,
        **kwargs
    ):
        """初始化 DashScope 模型

        Args:
            model_name: 模型名称
            api_key: DashScope API Key
            stream: 是否默认使用流式输出
        """
        super().__init__(model_name, api_key)

        # 设置 API Key
        if api_key:
            dashscope.api_key = api_key

        self.stream = stream

    async def __call__(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        stream: bool | None = None,
        **kwargs
    ) -> ChatResponse | AsyncGenerator[ChatResponse, None]:
        """调用 DashScope API"""

        # 准备请求参数
        params = {
            "model": self.model_name,
            "messages": messages,
            **kwargs
        }

        # 添加工具
        if tools:
            params["tools"] = tools

        # 是否流式
        use_stream = stream if stream is not None else self.stream

        if use_stream:
            # 流式调用
            return self._stream_call(params)
        else:
            # 非流式调用
            return await self._non_stream_call(params)

    async def _non_stream_call(self, params: dict) -> ChatResponse:
        """非流式调用"""
        response = await Generation.call(**params)

        # 提取内容
        content_blocks = self._parse_response(response)

        # 构造 ChatResponse
        return ChatResponse(
            id=response.request_id,
            content=content_blocks,
            created_at=datetime.now().isoformat(),
            usage={
                "prompt_tokens": response.usage.input_tokens,
                "completion_tokens": response.usage.output_tokens,
                "total_tokens": response.usage.total_tokens
            }
        )

    async def _stream_call(
        self,
        params: dict
    ) -> AsyncGenerator[ChatResponse, None]:
        """流式调用"""
        responses = Generation.call(stream=True, **params)

        accumulated_content = []

        for chunk in responses:
            if chunk.status_code == 200:
                # 解析块
                content_blocks = self._parse_response(chunk)

                # 累积内容
                accumulated_content.extend(content_blocks)

                # 判断是否为最后一块
                is_last = chunk.output.finish_reason is not None

                yield ChatResponse(
                    id=chunk.request_id,
                    content=accumulated_content.copy(),
                    created_at=datetime.now().isoformat(),
                    stream=True,
                    is_last=is_last,
                    usage={
                        "prompt_tokens": chunk.usage.input_tokens,
                        "completion_tokens": chunk.usage.output_tokens,
                        "total_tokens": chunk.usage.total_tokens
                    } if is_last else None
                )

    def _parse_response(self, response) -> list[ContentBlock]:
        """解析 DashScope 响应为内容块"""
        content_blocks = []

        # 解析文本内容
        if hasattr(response.output, "text"):
            content_blocks.append(
                TextBlock(type="text", text=response.output.text)
            )

        # 解析工具调用
        if hasattr(response.output, "tool_calls"):
            for tool_call in response.output.tool_calls:
                content_blocks.append(
                    ToolUseBlock(
                        type="tool_use",
                        id=tool_call.id,
                        name=tool_call.function.name,
                        input=json.loads(tool_call.function.arguments)
                    )
                )

        return content_blocks
```

**使用示例**:
```python
# 创建模型
model = DashScopeChatModel(
    model_name="qwen-max",
    api_key="your_api_key",
    stream=True  # 启用流式输出
)

# 非流式调用
messages = [
    {"role": "user", "content": "你好"}
]
response = await model(messages)
print(response.get_text_content())

# 流式调用
async for chunk in model(messages, stream=True):
    print(chunk.get_text_content(), end="", flush=True)
    if chunk.is_last:
        print(f"\n使用 Tokens: {chunk.usage['total_tokens']}")
```

#### 4.3.2 OpenAI 模型

**文件**: `_openai_model.py`

```python
from openai import AsyncOpenAI

class OpenAIChatModel(ChatModelBase):
    """OpenAI GPT 模型

    支持的模型:
    - gpt-4-turbo: GPT-4 Turbo
    - gpt-4: GPT-4
    - gpt-3.5-turbo: GPT-3.5 Turbo
    """

    def __init__(
        self,
        model_name: str = "gpt-4-turbo",
        api_key: str | None = None,
        api_base: str | None = None,
        **kwargs
    ):
        super().__init__(model_name, api_key, api_base)

        # 创建异步客户端
        self.client = AsyncOpenAI(
            api_key=api_key,
            base_url=api_base
        )

    async def __call__(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        stream: bool = False,
        **kwargs
    ) -> ChatResponse | AsyncGenerator[ChatResponse, None]:
        """调用 OpenAI API"""

        # 准备参数
        params = {
            "model": self.model_name,
            "messages": messages,
            "stream": stream,
            **kwargs
        }

        # 添加工具
        if tools:
            params["tools"] = [
                {"type": "function", "function": tool}
                for tool in tools
            ]

        if stream:
            return self._stream_call(params)
        else:
            return await self._non_stream_call(params)

    async def _non_stream_call(self, params: dict) -> ChatResponse:
        """非流式调用"""
        response = await self.client.chat.completions.create(**params)

        # 解析内容
        content_blocks = self._parse_choice(response.choices[0])

        return ChatResponse(
            id=response.id,
            content=content_blocks,
            created_at=datetime.fromtimestamp(response.created).isoformat(),
            usage={
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens,
                "total_tokens": response.usage.total_tokens
            }
        )

    async def _stream_call(
        self,
        params: dict
    ) -> AsyncGenerator[ChatResponse, None]:
        """流式调用"""
        stream = await self.client.chat.completions.create(**params)

        accumulated_content = []
        response_id = None

        async for chunk in stream:
            if not chunk.choices:
                continue

            delta = chunk.choices[0].delta

            # 保存 response ID
            if response_id is None:
                response_id = chunk.id

            # 解析增量内容
            if delta.content:
                accumulated_content.append(
                    TextBlock(type="text", text=delta.content)
                )

            if delta.tool_calls:
                for tool_call in delta.tool_calls:
                    accumulated_content.append(
                        ToolUseBlock(
                            type="tool_use",
                            id=tool_call.id,
                            name=tool_call.function.name,
                            input=json.loads(tool_call.function.arguments)
                        )
                    )

            # 判断是否结束
            is_last = chunk.choices[0].finish_reason is not None

            yield ChatResponse(
                id=response_id,
                content=accumulated_content.copy(),
                created_at=datetime.fromtimestamp(chunk.created).isoformat(),
                stream=True,
                is_last=is_last
            )
```

#### 4.3.3 Anthropic Claude 模型

**文件**: `_anthropic_model.py`

```python
from anthropic import AsyncAnthropic

class AnthropicChatModel(ChatModelBase):
    """Anthropic Claude 模型

    支持的模型:
    - claude-3-5-sonnet-20241022: Claude 3.5 Sonnet
    - claude-3-opus-20240229: Claude 3 Opus
    - claude-3-sonnet-20240229: Claude 3 Sonnet
    """

    def __init__(
        self,
        model_name: str = "claude-3-5-sonnet-20241022",
        api_key: str | None = None,
        **kwargs
    ):
        super().__init__(model_name, api_key)

        self.client = AsyncAnthropic(api_key=api_key)

    async def __call__(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        stream: bool = False,
        system: str | None = None,
        **kwargs
    ) -> ChatResponse | AsyncGenerator[ChatResponse, None]:
        """调用 Anthropic API

        Note: Anthropic 的 system 消息需要单独传递
        """

        params = {
            "model": self.model_name,
            "messages": messages,
            "stream": stream,
            "max_tokens": kwargs.pop("max_tokens", 4096),
            **kwargs
        }

        # 添加 system 消息
        if system:
            params["system"] = system

        # 添加工具
        if tools:
            params["tools"] = tools

        if stream:
            return self._stream_call(params)
        else:
            return await self._non_stream_call(params)

    async def _non_stream_call(self, params: dict) -> ChatResponse:
        """非流式调用"""
        response = await self.client.messages.create(**params)

        # 解析内容块
        content_blocks = []
        for block in response.content:
            if block.type == "text":
                content_blocks.append(
                    TextBlock(type="text", text=block.text)
                )
            elif block.type == "tool_use":
                content_blocks.append(
                    ToolUseBlock(
                        type="tool_use",
                        id=block.id,
                        name=block.name,
                        input=block.input
                    )
                )
            elif block.type == "thinking":
                # Claude 支持思考块
                content_blocks.append(
                    ThinkingBlock(
                        type="thinking",
                        thinking=block.thinking
                    )
                )

        return ChatResponse(
            id=response.id,
            content=content_blocks,
            created_at=datetime.now().isoformat(),
            usage={
                "prompt_tokens": response.usage.input_tokens,
                "completion_tokens": response.usage.output_tokens,
                "total_tokens": response.usage.input_tokens + response.usage.output_tokens
            },
            metadata={"stop_reason": response.stop_reason}
        )

    async def _stream_call(
        self,
        params: dict
    ) -> AsyncGenerator[ChatResponse, None]:
        """流式调用"""
        async with self.client.messages.stream(**params) as stream:
            accumulated_content = []
            response_id = None

            async for event in stream:
                # 消息开始事件
                if event.type == "message_start":
                    response_id = event.message.id

                # 内容块增量事件
                elif event.type == "content_block_delta":
                    delta = event.delta

                    if delta.type == "text_delta":
                        accumulated_content.append(
                            TextBlock(type="text", text=delta.text)
                        )

                    elif delta.type == "input_json_delta":
                        # 工具调用的参数增量
                        pass  # 累积处理

                # 消息结束事件
                elif event.type == "message_delta":
                    is_last = True

                    yield ChatResponse(
                        id=response_id,
                        content=accumulated_content.copy(),
                        created_at=datetime.now().isoformat(),
                        stream=True,
                        is_last=is_last,
                        usage={
                            "prompt_tokens": event.usage.input_tokens,
                            "completion_tokens": event.usage.output_tokens,
                            "total_tokens": event.usage.input_tokens + event.usage.output_tokens
                        }
                    )
```

### 4.4 流式处理机制

#### 4.4.1 流式输出的优势

1. **即时反馈**: 用户可以立即看到生成的内容
2. **降低延迟感**: 无需等待完整响应
3. **更好的用户体验**: 类似打字效果

#### 4.4.2 流式处理实现

```python
async def handle_stream_response(model, messages):
    """处理流式响应的示例"""

    print("Assistant: ", end="", flush=True)

    full_content = ""
    final_response = None

    async for chunk in model(messages, stream=True):
        # 提取新增文本
        current_content = chunk.get_text_content()
        new_content = current_content[len(full_content):]

        # 打印新增部分
        print(new_content, end="", flush=True)

        # 更新累积内容
        full_content = current_content

        # 保存最终响应
        if chunk.is_last:
            final_response = chunk

    print()  # 换行

    # 返回完整响应
    return final_response
```

#### 4.4.3 流式工具调用

某些模型（如 Anthropic Claude）支持流式工具调用：

```python
async def handle_stream_with_tools(model, messages, tools):
    """处理带工具的流式响应"""

    accumulated_tool_calls = []

    async for chunk in model(messages, tools=tools, stream=True):
        # 检查工具调用
        tool_uses = chunk.content

        for block in tool_uses:
            if block["type"] == "tool_use":
                # 流式累积工具调用
                accumulated_tool_calls.append(block)

        # 最后一块时执行工具
        if chunk.is_last:
            for tool_use in accumulated_tool_calls:
                result = await execute_tool(
                    tool_use["name"],
                    tool_use["input"]
                )
                print(f"工具 {tool_use['name']} 执行结果: {result}")
```

---

## 第五章: Message 与通信机制

**文件位置**: `src/agentscope/message/`

### 5.1 Msg 类设计

**文件**: `_message_base.py`

#### 5.1.1 核心结构

```python
from dataclasses import dataclass, field
from datetime import datetime
import shortuuid

@dataclass
class Msg:
    """消息类

    设计理念:
    1. 统一消息格式
    2. 支持多模态内容
    3. 可序列化
    4. 元数据扩展
    """

    name: str
    """发送者名称（Agent 名称或 "user"、"system"）"""

    content: str | list[ContentBlock]
    """消息内容：纯文本或内容块列表"""

    role: str
    """角色：user, assistant, system"""

    id: str = field(default_factory=lambda: shortuuid.uuid())
    """消息唯一标识"""

    metadata: dict | None = None
    """元数据：存储额外信息"""

    timestamp: str = field(
        default_factory=lambda: datetime.now().isoformat()
    )
    """时间戳"""

    invocation_id: str | None = None
    """调用标识：同一次 Agent 调用的消息共享此 ID"""

    def __post_init__(self):
        """初始化后处理"""
        # 如果 content 是字符串，转换为 TextBlock
        if isinstance(self.content, str):
            self.content = [TextBlock(type="text", text=self.content)]
```

#### 5.1.2 核心方法

```python
class Msg:
    # ... 属性定义 ...

    def get_text_content(self) -> str:
        """提取所有文本内容

        Returns:
            拼接后的文本字符串
        """
        if isinstance(self.content, str):
            return self.content

        text_parts = []
        for block in self.content:
            if block["type"] == "text":
                text_parts.append(block["text"])
            elif block["type"] == "thinking":
                text_parts.append(block["thinking"])

        return "".join(text_parts)

    def get_content_blocks(
        self,
        block_type: str | None = None
    ) -> list[ContentBlock]:
        """获取内容块

        Args:
            block_type: 块类型过滤（如 "tool_use"、"image" 等）
                      如果为 None，返回所有块

        Returns:
            内容块列表
        """
        if isinstance(self.content, str):
            return [TextBlock(type="text", text=self.content)]

        if block_type is None:
            return self.content

        return [
            block for block in self.content
            if block.get("type") == block_type
        ]

    def to_dict(self) -> dict:
        """序列化为字典

        Returns:
            字典表示
        """
        return {
            "id": self.id,
            "name": self.name,
            "content": self._serialize_content(),
            "role": self.role,
            "metadata": self.metadata,
            "timestamp": self.timestamp,
            "invocation_id": self.invocation_id
        }

    def _serialize_content(self) -> list[dict]:
        """序列化内容"""
        if isinstance(self.content, str):
            return [{"type": "text", "text": self.content}]

        return [
            self._serialize_block(block)
            for block in self.content
        ]

    def _serialize_block(self, block: ContentBlock) -> dict:
        """序列化单个内容块"""
        # 直接返回字典表示
        if isinstance(block, dict):
            return block

        # 如果是对象，调用其序列化方法
        if hasattr(block, "to_dict"):
            return block.to_dict()

        # 否则转换为字典
        return dict(block)

    @classmethod
    def from_dict(cls, data: dict) -> "Msg":
        """从字典反序列化

        Args:
            data: 字典数据

        Returns:
            Msg 对象
        """
        # 反序列化内容块
        content = [
            cls._deserialize_block(block)
            for block in data["content"]
        ]

        return cls(
            id=data["id"],
            name=data["name"],
            content=content,
            role=data["role"],
            metadata=data.get("metadata"),
            timestamp=data["timestamp"],
            invocation_id=data.get("invocation_id")
        )

    @classmethod
    def _deserialize_block(cls, block_dict: dict) -> ContentBlock:
        """反序列化内容块"""
        block_type = block_dict["type"]

        # 根据类型创建对应的块对象
        if block_type == "text":
            return TextBlock(**block_dict)
        elif block_type == "image":
            return ImageBlock(**block_dict)
        elif block_type == "audio":
            return AudioBlock(**block_dict)
        elif block_type == "video":
            return VideoBlock(**block_dict)
        elif block_type == "tool_use":
            return ToolUseBlock(**block_dict)
        elif block_type == "tool_result":
            return ToolResultBlock(**block_dict)
        elif block_type == "thinking":
            return ThinkingBlock(**block_dict)
        else:
            # 未知类型，返回原始字典
            return block_dict
```

### 5.2 内容块系统

**文件**: `_message_block.py`

#### 5.2.1 TextBlock - 文本块

```python
from typing import TypedDict

class TextBlock(TypedDict):
    """文本内容块

    最基础的内容类型
    """
    type: str  # "text"
    text: str  # 文本内容

# 使用示例
text_block = TextBlock(
    type="text",
    text="Hello, world!"
)
```

#### 5.2.2 ImageBlock - 图片块

```python
class ImageSource(TypedDict):
    """图片来源"""
    type: str  # "url" 或 "base64"

class URLSource(ImageSource):
    """URL 图片源"""
    type: str  # "url"
    url: str   # 图片 URL

class Base64Source(ImageSource):
    """Base64 图片源"""
    type: str        # "base64"
    media_type: str  # 媒体类型（如 "image/png"）
    data: str        # Base64 编码数据

class ImageBlock(TypedDict):
    """图片内容块"""
    type: str          # "image"
    source: ImageSource  # 图片来源

# 使用示例 - URL 图片
image_url = ImageBlock(
    type="image",
    source=URLSource(
        type="url",
        url="https://example.com/image.jpg"
    )
)

# 使用示例 - Base64 图片
with open("image.png", "rb") as f:
    image_data = base64.b64encode(f.read()).decode()

image_base64 = ImageBlock(
    type="image",
    source=Base64Source(
        type="base64",
        media_type="image/png",
        data=image_data
    )
)
```

#### 5.2.3 AudioBlock - 音频块

```python
class AudioBlock(TypedDict):
    """音频内容块"""
    type: str          # "audio"
    source: dict       # 音频来源（URL 或 Base64）

# 使用示例
audio_block = AudioBlock(
    type="audio",
    source={
        "type": "base64",
        "media_type": "audio/wav",
        "data": "base64_audio_data..."
    }
)
```

#### 5.2.4 ToolUseBlock - 工具调用块

```python
class ToolUseBlock(TypedDict):
    """工具调用块

    表示 LLM 决定调用某个工具
    """
    type: str       # "tool_use"
    id: str         # 工具调用 ID
    name: str       # 工具名称
    input: dict     # 工具参数（JSON 对象）

# 使用示例
tool_use = ToolUseBlock(
    type="tool_use",
    id="call_123",
    name="execute_python_code",
    input={
        "code": "print(2 + 2)"
    }
)
```

#### 5.2.5 ToolResultBlock - 工具结果块

```python
class ToolResultBlock(TypedDict):
    """工具结果块

    表示工具执行的结果
    """
    type: str              # "tool_result"
    id: str                # 工具调用 ID（对应 ToolUseBlock 的 id）
    output: str | list[ContentBlock]  # 工具输出

# 使用示例
tool_result = ToolResultBlock(
    type="tool_result",
    id="call_123",
    output="4\n"  # 或者是更复杂的内容块列表
)
```

#### 5.2.6 ThinkingBlock - 思考块

```python
class ThinkingBlock(TypedDict):
    """思考块

    某些模型（如 Claude）支持展示其思考过程
    """
    type: str       # "thinking"
    thinking: str   # 思考内容

# 使用示例
thinking = ThinkingBlock(
    type="thinking",
    thinking="Let me break this down step by step..."
)
```

### 5.3 多模态支持

#### 5.3.1 构造多模态消息

```python
# 示例：包含文本、图片和音频的消息
multimodal_msg = Msg(
    name="user",
    content=[
        TextBlock(
            type="text",
            text="请分析这张图片和这段音频"
        ),
        ImageBlock(
            type="image",
            source=URLSource(
                type="url",
                url="https://example.com/chart.png"
            )
        ),
        AudioBlock(
            type="audio",
            source={
                "type": "base64",
                "media_type": "audio/wav",
                "data": audio_base64_data
            }
        )
    ],
    role="user"
)
```

#### 5.3.2 处理多模态内容

```python
def process_multimodal_message(msg: Msg):
    """处理多模态消息"""

    # 提取文本
    text = msg.get_text_content()
    print(f"文本: {text}")

    # 提取图片
    images = msg.get_content_blocks("image")
    for img in images:
        if img["source"]["type"] == "url":
            print(f"图片 URL: {img['source']['url']}")
        elif img["source"]["type"] == "base64":
            print(f"图片类型: {img['source']['media_type']}")
            # 处理 Base64 数据

    # 提取音频
    audios = msg.get_content_blocks("audio")
    for audio in audios:
        print(f"音频: {audio['source']}")
```

### 5.4 消息序列化

#### 5.4.1 JSON 序列化

```python
import json

# 序列化消息
msg = Msg(
    name="agent1",
    content="Hello",
    role="assistant"
)

msg_dict = msg.to_dict()
msg_json = json.dumps(msg_dict, ensure_ascii=False, indent=2)

print(msg_json)
# {
#   "id": "abc123",
#   "name": "agent1",
#   "content": [{"type": "text", "text": "Hello"}],
#   "role": "assistant",
#   "metadata": null,
#   "timestamp": "2025-10-07T10:30:00",
#   "invocation_id": null
# }

# 反序列化
loaded_dict = json.loads(msg_json)
loaded_msg = Msg.from_dict(loaded_dict)

assert loaded_msg.name == msg.name
assert loaded_msg.get_text_content() == msg.get_text_content()
```

#### 5.4.2 保存和加载消息历史

```python
class MessageHistory:
    """消息历史管理"""

    def __init__(self, file_path: str):
        self.file_path = file_path
        self.messages: list[Msg] = []

    def add(self, msg: Msg):
        """添加消息"""
        self.messages.append(msg)

    def save(self):
        """保存到文件"""
        with open(self.file_path, "w", encoding="utf-8") as f:
            json.dump(
                [msg.to_dict() for msg in self.messages],
                f,
                ensure_ascii=False,
                indent=2
            )

    def load(self):
        """从文件加载"""
        with open(self.file_path, "r", encoding="utf-8") as f:
            data = json.load(f)
            self.messages = [Msg.from_dict(d) for d in data]

    def get_all(self) -> list[Msg]:
        """获取所有消息"""
        return self.messages

# 使用示例
history = MessageHistory("conversation.json")

history.add(Msg("user", "Hello", "user"))
history.add(Msg("assistant", "Hi there!", "assistant"))
history.add(Msg("user", "How are you?", "user"))

history.save()

# 稍后加载
new_history = MessageHistory("conversation.json")
new_history.load()

for msg in new_history.get_all():
    print(f"{msg.name}: {msg.get_text_content()}")
```

---

## 第六章: Tool 工具系统

**文件位置**: `src/agentscope/tool/`

### 6.1 Toolkit 管理器设计

Toolkit 是工具管理的核心，负责注册、组织和调用所有工具函数。

**核心职责**:
- 注册工具函数（Python 函数、MCP 工具）
- 生成 JSON Schema 供 LLM 使用
- 管理工具分组和激活状态
- 执行工具调用（支持同步/异步/流式）

**完整实现详见第二部分文档第 3.7 节**

### 6.2 工具注册与 JSON Schema

工具通过装饰器或直接注册方式添加到 Toolkit，系统自动从函数签名和文档字符串生成 JSON Schema。

**示例**:
```python
def my_tool(param1: str, param2: int = 10) -> ToolResponse:
    """工具描述

    Args:
        param1: 参数1描述
        param2: 参数2描述
    """
    return ToolResponse(...)

toolkit.register_tool_function(my_tool)
# 自动生成 JSON Schema 供 LLM 调用
```

### 6.3 工具组管理

工具可以组织为逻辑组，动态启用/禁用：

```python
toolkit.create_tool_group("web", "Web tools", active=False)
toolkit.register_tool_function(fetch_url, group_name="web")

# 根据需要激活
toolkit.update_tool_groups(["web"], active=True)
```

### 6.4 MCP 工具集成

支持通过 MCP 协议集成外部工具服务器：

```python
mcp_client = HttpStatelessClient(url="...")
await toolkit.register_mcp_client(mcp_client)
# MCP 工具自动注册为普通工具函数
```

### 6.5 内置工具

AgentScope 提供了代码执行、文件操作、多模态生成等内置工具。详细实现参见第二部分文档。

---

## 第七章: Memory 记忆系统

**文件位置**: `src/agentscope/memory/`

### 7.1 短期记忆（InMemoryMemory）

存储当前对话的消息历史，Agent 通过短期记忆维持上下文。

**核心方法**:
```python
memory = InMemoryMemory()
await memory.add(msg)              # 添加消息
messages = await memory.get_memory()  # 获取所有消息
await memory.clear()               # 清空记忆
```

### 7.2 长期记忆（Mem0LongTermMemory）

通过 Mem0 平台实现持久化记忆，支持跨会话的知识保存和检索。

**三种管理模式**:
1. **agent_control**: Agent 通过工具函数主动控制
2. **static_control**: 每次对话自动检索和记录
3. **both**: 自动管理 + Agent 可主动调用

### 7.3 记忆管理策略

**示例**:
```python
agent = ReActAgent(
    long_term_memory=Mem0LongTermMemory(...),
    long_term_memory_mode="both",  # 自动 + 主动
    ...
)
# Agent 每次 reply() 开始时自动检索相关记忆
# Agent 也可以主动调用 retrieve_from_memory 工具
```

---

## 第八章: Formatter 格式化系统

**文件位置**: `src/agentscope/formatter/`

### 8.1 FormatterBase 基类

Formatter 负责将 AgentScope 的 Msg 对象转换为不同 LLM API 所需的格式。

**核心方法**:
```python
async def format(
    self,
    msgs: list[Msg],
    tools: list[dict] | None = None
) -> list[dict]:
    """格式化消息为 API 格式"""
```

### 8.2 各厂商 Formatter

不同 LLM 提供商有不同的消息格式：

**DashScopeChatFormatter**: 阿里云通义千问格式
**OpenAIChatFormatter**: OpenAI GPT 格式
**AnthropicChatFormatter**: Anthropic Claude 格式（system 消息单独传递）
**GeminiChatFormatter**: Google Gemini 格式

### 8.3 截断策略

当消息历史过长时，Formatter 可以使用截断策略：

```python
formatter = DashScopeChatFormatter(
    max_length=4000,  # 最大 token 数
    truncation_mode="keep_recent"  # 保留最近的消息
)
```

---

## 第九章: Pipeline 工作流编排

**文件位置**: `src/agentscope/pipeline/`

### 9.1 MsgHub 消息中心

MsgHub 实现发布-订阅模式，Agent 之间自动广播消息：

```python
async with MsgHub([agent1, agent2, agent3], announcement=topic_msg) as hub:
    await agent1(msg)  # agent1 的回复自动广播给 agent2 和 agent3
    await agent2()      # agent2 已经"听到"了 agent1 的回复
```

### 9.2 Pipeline 模式

**顺序执行**:
```python
result = await sequential_pipeline([agent1, agent2, agent3], initial_msg)
# agent1 → agent2 → agent3
```

**并行执行**:
```python
results = await fanout_pipeline([agent1, agent2, agent3], query_msg)
# agent1 ║ agent2 ║ agent3 (并行)
```

### 9.3 复杂工作流设计

组合使用 MsgHub 和 Pipeline 可以构建复杂的多 Agent 协作流程。详见第二部分文档第 3.7 节。

---

## 第十章: Plan 规划系统

**文件位置**: `src/agentscope/plan/`

### 10.1 PlanNotebook 设计

PlanNotebook 帮助 Agent 管理复杂任务的规划和执行：

**核心功能**:
- 创建多层级任务计划（Plan 包含多个 SubTask）
- 追踪任务执行状态（todo, in_progress, done, abandoned）
- 为 Agent 生成当前任务提示
- 提供规划相关的工具函数

### 10.2 任务分解机制

Agent 可以调用工具函数创建计划：

```python
# Agent 调用 create_plan 工具
{
    "name": "开发 Web 爬虫",
    "description": "...",
    "expected_outcome": "...",
    "subtasks": [
        {"name": "研究库", "description": "...", "expected_outcome": "..."},
        {"name": "设计架构", "description": "...", "expected_outcome": "..."},
        {"name": "实现代码", "description": "...", "expected_outcome": "..."},
    ]
}
```

### 10.3 计划执行追踪

PlanNotebook 在每次 Agent 推理时提供当前任务提示：

```python
plan_notebook = PlanNotebook()
agent = ReActAgent(plan_notebook=plan_notebook, ...)

# Agent 会自动获得类似这样的提示：
# Current Plan: 开发 Web 爬虫
# Current Subtask (2/3): 设计架构
# - Description: ...
# - Expected Outcome: ...
# Please work on this subtask. When finished, call finish_subtask().
```

详细实现参见第二部分文档第 3.8 节。

---

## 第十一章: RAG 检索增强

**文件位置**: `src/agentscope/rag/`

### 11.1 KnowledgeBase 设计

KnowledgeBase 提供基于向量检索的知识增强能力：

**核心组件**:
- **Document**: 文档模型（内容 + 向量 + 元数据）
- **EmbeddingModel**: 文本向量化模型
- **VDBStore**: 向量数据库存储（如 Qdrant）
- **Reader**: 文档读取器（Text, PDF, Image）

### 11.2 文档处理

文档经过读取、分块、向量化后存入知识库：

```python
kb = SimpleKnowledgeBase(
    embedding_model=DashScopeTextEmbedding(...),
    store=QdrantStore(...)
)

# 添加文档
await kb.add_files(["manual.pdf", "faq.txt"])
```

### 11.3 向量检索

Agent 查询时自动检索相关文档：

```python
agent = ReActAgent(
    knowledge=[kb1, kb2],
    enable_rewrite_query=True,  # 查询重写优化
    ...
)

# Agent 每次 reply() 开始时：
# 1. 从知识库检索相关文档
# 2. 将文档添加到记忆
# 3. LLM 基于文档生成回复
```

详细实现参见第二部分文档第 3.9 节。

---

## 第十二章: MCP 协议集成

**文件位置**: `src/agentscope/mcp/`

### 12.1 MCP 客户端架构

MCP (Model Context Protocol) 允许 Agent 连接外部工具服务器。

**支持的传输方式**:
- **HTTP Stateless**: 无状态 HTTP 请求
- **HTTP Stateful**: 有状态 HTTP 会话
- **StdIO**: 通过标准输入/输出与本地进程通信

### 12.2 HTTP 客户端

```python
from agentscope.mcp import HttpStatelessClient

client = HttpStatelessClient(url="https://mcp.example.com")
await toolkit.register_mcp_client(client)
# MCP 工具自动注册到 toolkit
```

### 12.3 StdIO 客户端

用于启动本地 MCP 服务器进程：

```python
from agentscope.mcp import StdIOStatefulClient

async with StdIOStatefulClient(
    command="python",
    args=["mcp_server.py"]
) as client:
    result = await client.call_tool("analyze", {"text": "..."})
```

详细实现参见第二部分文档第 3.10 节。

---

## 第十三章: 辅助模块

### 13.1 Embedding 嵌入模型

**文件位置**: `src/agentscope/embedding/`

支持文本向量化，用于 RAG 等场景：

```python
embedding_model = DashScopeTextEmbedding(model_name="text-embedding-v1")
embeddings = await embedding_model(["text1", "text2"])
```

**支持缓存**减少 API 调用：
```python
from agentscope.embedding import FileEmbeddingCache

cache = FileEmbeddingCache(cache_dir="./cache")
embedding_model = DashScopeTextEmbedding(cache=cache)
```

### 13.2 Token 计数

**文件位置**: `src/agentscope/token/`

用于估算 token 使用和成本：

```python
from agentscope.token import OpenAITokenCounter

counter = OpenAITokenCounter(model_name="gpt-4")
token_count = counter.count(messages)
cost = counter.estimate_cost(token_count)
```

### 13.3 Session 会话管理

**文件位置**: `src/agentscope/session/`

保存和恢复 Agent 状态：

```python
from agentscope.session import JsonSession

session = JsonSession(session_dir="./sessions", session_id="user_001")
session.save_agent(agent)

# 稍后加载
loaded_agent = session.load_agent("agent_name")
```

### 13.4 Tracing 追踪系统

**文件位置**: `src/agentscope/tracing/`

基于 OpenTelemetry 的分布式追踪：

```python
from agentscope.tracing import setup_tracing

setup_tracing(
    endpoint="http://localhost:4317",  # Jaeger/Phoenix
    service_name="my_agent_app"
)
# 所有 Agent、Model、Tool 调用自动追踪
```

---

## 第十四章: 执行流程深度剖析

### 14.1 ReActAgent 完整执行流

完整的执行流程已在第三章详细描述。核心流程：

```
1. 用户输入 → UserAgent
2. ReActAgent.__call__()
   ├─ pre_reply 钩子
   ├─ reply()
   │  ├─ memory.add(msg)
   │  ├─ long_term_memory.retrieve() [可选]
   │  ├─ knowledge.retrieve() [可选]
   │  ├─ plan_notebook.get_hint() [可选]
   │  └─ ReAct 循环（最多 max_iters 次）
   │     ├─ _reasoning()
   │     │  ├─ formatter.format()
   │     │  ├─ model()
   │     │  └─ 提取 tool_calls
   │     ├─ _acting(tool_calls) [如果有]
   │     │  └─ toolkit.call_tool_function()
   │     └─ 继续循环或结束
   ├─ post_reply 钩子
   ├─ _broadcast_to_subscribers()
   └─ print()
3. 返回响应
```

### 14.2 多 Agent 协作流程

通过 MsgHub 实现自动消息广播：

```
MsgHub([agent1, agent2, agent3])
├─ 设置订阅关系
│  agent1 订阅 [agent2, agent3]
│  agent2 订阅 [agent1, agent3]
│  agent3 订阅 [agent1, agent2]
└─ 执行流程
   agent1(msg) → response1
   └─ 自动广播 → agent2.observe(response1)
                 agent3.observe(response1)
```

### 14.3 工具调用流程

详见第六章 6.6 节。核心步骤：

1. LLM 返回 ToolUseBlock
2. Agent 提取工具调用
3. Toolkit 查找并执行工具
4. 结果包装为 ToolResultBlock
5. 添加到记忆，继续下一轮推理

---

## 第十五章: 扩展开发指南

### 15.1 自定义 Agent

**继承 AgentBase**:
```python
class MyAgent(AgentBase):
    async def reply(self, msg: Msg) -> Msg:
        # 自定义逻辑
        return Msg(self.name, "response", "assistant")
```

**继承 ReActAgent**:
```python
class MyEnhancedAgent(ReActAgent):
    async def _reasoning(self, *args, **kwargs):
        # 自定义推理过程
        return await super()._reasoning(*args, **kwargs)
```

### 15.2 自定义工具函数

```python
def my_tool(param1: str, param2: int) -> ToolResponse:
    """工具描述

    Args:
        param1: 参数描述
        param2: 参数描述
    """
    result = process(param1, param2)
    return ToolResponse(
        content=[TextBlock(type="text", text=result)]
    )

toolkit.register_tool_function(my_tool)
```

**异步工具**:
```python
async def async_tool(url: str) -> ToolResponse:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            text = await response.text()
    return ToolResponse(content=[TextBlock(type="text", text=text)])
```

**流式工具**:
```python
async def streaming_tool(data: str) -> AsyncGenerator[ToolResponse, None]:
    for chunk in process_stream(data):
        yield ToolResponse(
            content=[TextBlock(type="text", text=chunk)],
            stream=True,
            is_last=False
        )
    yield ToolResponse(
        content=[TextBlock(type="text", text="Complete")],
        stream=True,
        is_last=True
    )
```

### 15.3 自定义组件

**自定义 Memory**:
```python
class MyMemory(MemoryBase):
    async def add(self, memories): ...
    async def get_memory(self): ...
    async def clear(self): ...
```

**自定义 Formatter**:
```python
class MyFormatter(FormatterBase):
    async def format(self, msgs, tools=None):
        # 转换为自定义 API 格式
        return formatted_messages
```

**自定义 KnowledgeBase**:
```python
class MyKnowledgeBase(KnowledgeBase):
    async def add_documents(self, documents): ...
    async def retrieve(self, query, limit, score_threshold): ...
```

详细示例参见第二部分文档第 7 章。

---

## 第十六章: 最佳实践与优化

### 16.1 性能优化

**并行执行**:
```python
# 并行 Agent 调用
results = await asyncio.gather(
    agent1(msg), agent2(msg), agent3(msg)
)

# 并行工具调用（ReActAgent 自动支持）
agent = ReActAgent(parallel_tool_calls=True, ...)
```

**使用缓存**:
```python
# Embedding 缓存
embedding_model = DashScopeTextEmbedding(cache=FileEmbeddingCache(...))

# 减少不必要的 API 调用
```

**流式输出**:
```python
# 提升用户体验
model = DashScopeChatModel(stream=True)
agent = ReActAgent(model=model, ...)
```

### 16.2 错误处理

**捕获特定异常**:
```python
from agentscope.exception import ToolNotFoundError

try:
    result = await agent(msg)
except ToolNotFoundError as e:
    logger.error(f"Tool '{e.tool_name}' not found")
    # 处理
```

**优雅降级**:
```python
async def robust_call(agent, msg, max_retries=3):
    for attempt in range(max_retries):
        try:
            return await agent(msg)
        except Exception as e:
            if attempt == max_retries - 1:
                return Msg("system", f"Error: {e}", "system")
            await asyncio.sleep(2 ** attempt)
```

### 16.3 测试策略

**Mock Model**:
```python
class MockModel(ChatModelBase):
    def __init__(self, responses):
        self.responses = responses
        self.call_count = 0

    async def __call__(self, messages, **kwargs):
        response = self.responses[self.call_count]
        self.call_count += 1
        return ChatResponse(
            content=[TextBlock(type="text", text=response)],
            ...
        )
```

**状态序列化测试**:
```python
# 测试 Agent 状态保存和恢复
state = agent.state_dict()
new_agent = ReActAgent(...)
new_agent.load_state_dict(state)
assert agent.memory.content == new_agent.memory.content
```

**工具测试**:
```python
# 单独测试工具函数
result = await my_tool(param1="test", param2=10)
assert isinstance(result, ToolResponse)
assert "expected" in result.content[0]["text"]
```

更多最佳实践参见第二部分文档第 8 章。

---

## 附录

### A.1 常用导入

```python
# 核心组件
from agentscope.agent import ReActAgent, UserAgent
from agentscope.message import Msg
from agentscope.model import DashScopeChatModel, OpenAIChatModel
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.tool import Toolkit, ToolResponse
from agentscope.pipeline import MsgHub, sequential_pipeline, fanout_pipeline

# 高级功能
from agentscope.plan import PlanNotebook
from agentscope.rag import SimpleKnowledgeBase
from agentscope.mcp import HttpStatelessClient
from agentscope.embedding import DashScopeTextEmbedding
```

### A.2 快速开始模板

```python
import agentscope
from agentscope.agent import ReActAgent
from agentscope.message import Msg
from agentscope.model import DashScopeChatModel
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.tool import Toolkit

# 初始化
agentscope.init(project="my_project", logging_level="INFO")

# 创建 Agent
agent = ReActAgent(
    name="assistant",
    sys_prompt="You are a helpful assistant",
    model=DashScopeChatModel(
        model_name="qwen-max",
        api_key="your_api_key"
    ),
    memory=InMemoryMemory(),
    formatter=DashScopeChatFormatter(),
    toolkit=Toolkit()
)

# 使用 Agent
msg = Msg("user", "Hello!", "user")
response = await agent(msg)
print(response.get_text_content())
```

### A.3 参考资源

**官方资源**:
- GitHub: https://github.com/agentscope-ai/agentscope
- 文档: https://doc.agentscope.io/
- 论文: https://arxiv.org/abs/2508.16279

**社区**:
- Discord: https://discord.gg/eYMpfnkG8h

---

## 文档完成

本文档全面分析了 AgentScope 项目的：
- ✅ 项目概览与核心理念（第一章）
- ✅ 核心架构设计（第二章）
- ✅ Agent 模块深度解析（第三章）
- ✅ Model 模块详细分析（第四章）
- ✅ Message 与通信机制（第五章）
- ✅ Tool 工具系统（第六章）
- ✅ Memory 记忆系统（第七章）
- ✅ Formatter 格式化系统（第八章）
- ✅ Pipeline 工作流编排（第九章）
- ✅ Plan 规划系统（第十章）
- ✅ RAG 检索增强（第十一章）
- ✅ MCP 协议集成（第十二章）
- ✅ 辅助模块（第十三章）
- ✅ 执行流程深度剖析（第十四章）
- ✅ 扩展开发指南（第十五章）
- ✅ 最佳实践与优化（第十六章）

**文档说明**:
- 本文档整合了项目的完整架构分析
- 第一至第五章提供了详细的代码实现和设计理念
- 第六至第十六章提供了核心概念和要点总结
- 更详细的实现细节和代码示例请参考第二部分文档（`AgentScope项目架构分析文档-第二部分.md`）

**推荐阅读顺序**:
1. 第一章：了解项目背景和设计理念
2. 第二章：理解整体架构
3. 第三章：深入学习 Agent 实现
4. 第四、五章：掌握 Model 和 Message 机制
5. 第六至十二章：学习各核心模块
6. 第十三至十六章：应用实践和优化

希望这份文档能帮助您深入理解和使用 AgentScope 框架！

---

*文档版本: 2.0 完整版*
*最后更新: 2025-10-07*
*项目版本: AgentScope 1.0.4*

