# AgentScope 项目架构分析文档 - 第二部分

> 本文档是《AgentScope项目架构分析文档.md》的续篇，包含 Pipeline、Plan、RAG、MCP 等模块的详细分析

---

## 目录

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
- [9. 项目总结](#9-项目总结)

---

### 3.7 Pipeline 模块

**文件位置**: `src/agentscope/pipeline/`

#### 3.7.1 文件结构

```
pipeline/
├── __init__.py          # 导出接口
├── _msghub.py          # MsgHub 消息中心
├── _class.py           # Pipeline 类
└── _functional.py      # 函数式 Pipeline
```

#### 3.7.2 MsgHub 类

**文件**: `_msghub.py`

MsgHub 是一个发布-订阅模式的消息中心，用于管理多 Agent 之间的消息广播。

**核心设计**:
```python
class MsgHub:
    """消息中心

    特性:
    - 发布-订阅模式
    - 自动消息广播
    - 动态参与者管理
    - 上下文管理器支持
    """

    def __init__(
        self,
        participants: list[AgentBase],
        announcement: Msg | list[Msg] | None = None,
        enable_auto_broadcast: bool = True,
        name: str | None = None
    ):
        """初始化消息中心

        Args:
            participants: 参与的 Agent 列表
            announcement: 初始公告消息
            enable_auto_broadcast: 是否自动广播消息
            name: 消息中心名称
        """
        self.participants = participants
        self.announcement = announcement
        self.enable_auto_broadcast = enable_auto_broadcast
        self.name = name or f"msghub_{uuid.uuid4().hex[:8]}"
```

**上下文管理**:
```python
async def __aenter__(self):
    """进入上下文时设置订阅关系"""
    self._reset_subscriber()

    # 广播初始公告
    if self.announcement:
        await self.broadcast(self.announcement)

    return self

async def __aexit__(self, *args, **kwargs):
    """退出上下文时清理订阅"""
    for agent in self.participants:
        agent.remove_subscribers(self.name)
```

**核心方法**:

1. **_reset_subscriber() - 重置订阅关系**
```python
def _reset_subscriber(self) -> None:
    """为每个参与者设置订阅关系

    每个 Agent 订阅其他所有 Agent 的消息
    """
    for agent in self.participants:
        # 获取其他参与者
        other_participants = [
            p for p in self.participants
            if p.name != agent.name
        ]

        # 设置订阅
        agent.set_subscribers(self.name, other_participants)
```

2. **broadcast() - 广播消息**
```python
async def broadcast(
    self,
    msg: Msg | list[Msg]
) -> None:
    """向所有参与者广播消息

    Args:
        msg: 要广播的消息
    """
    if not isinstance(msg, list):
        msg = [msg]

    # 让所有参与者观察消息
    await asyncio.gather(*[
        agent.observe(msg)
        for agent in self.participants
    ])
```

3. **add() - 添加参与者**
```python
def add(self, agent: AgentBase) -> None:
    """动态添加参与者"""
    if agent not in self.participants:
        self.participants.append(agent)
        self._reset_subscriber()
```

4. **delete() - 移除参与者**
```python
def delete(self, agent: AgentBase) -> None:
    """动态移除参与者"""
    if agent in self.participants:
        self.participants.remove(agent)
        agent.remove_subscribers(self.name)
        self._reset_subscriber()
```

**使用示例**:
```python
# 创建 Agent
agent1 = ReActAgent(name="Agent1", ...)
agent2 = ReActAgent(name="Agent2", ...)
agent3 = ReActAgent(name="Agent3", ...)

# 使用 MsgHub
async with MsgHub(
    participants=[agent1, agent2, agent3],
    announcement=Msg("host", "Let's discuss this topic", "system")
) as hub:
    # Agent1 发言，自动广播给 Agent2 和 Agent3
    msg1 = await agent1(Msg("user", "What do you think?", "user"))

    # Agent2 发言，自动广播给 Agent1 和 Agent3
    msg2 = await agent2()

    # 动态添加新参与者
    agent4 = ReActAgent(name="Agent4", ...)
    hub.add(agent4)

    # Agent3 发言，自动广播给所有参与者（包括 Agent4）
    msg3 = await agent3()

    # 移除参与者
    hub.delete(agent1)

    # 手动广播消息
    await hub.broadcast(Msg("host", "Thank you all!", "system"))

# 退出上下文后，订阅关系自动清理
```

#### 3.7.3 Pipeline 类

**文件**: `_class.py`

Pipeline 提供了多种 Agent 编排方式。

**1. SequentialPipeline - 顺序执行**

```python
class SequentialPipeline:
    """顺序执行 Agent 列表

    特性:
    - 按顺序执行每个 Agent
    - 前一个 Agent 的输出作为下一个的输入
    - 返回最后一个 Agent 的输出
    """

    def __init__(self, agents: list[AgentBase]):
        """初始化顺序 Pipeline

        Args:
            agents: Agent 列表
        """
        self.agents = agents

    async def __call__(
        self,
        msg: Msg | list[Msg] | None = None
    ) -> Msg | list[Msg] | None:
        """执行 Pipeline

        Args:
            msg: 初始消息

        Returns:
            最后一个 Agent 的输出
        """
        current_msg = msg

        for agent in self.agents:
            current_msg = await agent(current_msg)

        return current_msg
```

**使用示例**:
```python
# 创建 Pipeline
pipeline = SequentialPipeline([agent1, agent2, agent3])

# 执行
result = await pipeline(Msg("user", "Start", "user"))
# agent1 处理 -> agent2 处理 -> agent3 处理 -> 返回 result
```

**2. FanoutPipeline - 并行执行（扇出）**

```python
class FanoutPipeline:
    """并行执行 Agent 列表（扇出模式）

    特性:
    - 所有 Agent 并行执行
    - 所有 Agent 接收相同的输入
    - 返回所有 Agent 的输出列表
    """

    def __init__(
        self,
        agents: list[AgentBase],
        enable_gather: bool = True
    ):
        """初始化扇出 Pipeline

        Args:
            agents: Agent 列表
            enable_gather: 是否等待所有 Agent 完成
        """
        self.agents = agents
        self.enable_gather = enable_gather

    async def __call__(
        self,
        msg: Msg | list[Msg] | None = None,
        **kwargs
    ) -> list[Msg]:
        """执行 Pipeline

        Args:
            msg: 输入消息（所有 Agent 接收相同消息）

        Returns:
            所有 Agent 的输出列表
        """
        if self.enable_gather:
            # 等待所有 Agent 完成
            results = await asyncio.gather(*[
                agent(msg, **kwargs)
                for agent in self.agents
            ])
        else:
            # 不等待，直接启动所有任务
            tasks = [
                asyncio.create_task(agent(msg, **kwargs))
                for agent in self.agents
            ]
            results = tasks

        return results
```

**使用示例**:
```python
# 创建并行 Pipeline
pipeline = FanoutPipeline([agent1, agent2, agent3])

# 并行执行
results = await pipeline(Msg("user", "Analyze this", "user"))
# [agent1 输出, agent2 输出, agent3 输出]

# 各 Agent 可以给出不同的分析结果
```

#### 3.7.4 函数式 Pipeline

**文件**: `_functional.py`

提供函数式的 Pipeline 接口。

**1. sequential_pipeline()**
```python
async def sequential_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None
) -> Msg | list[Msg] | None:
    """顺序执行 Agent

    等价于 SequentialPipeline(...)(msg)
    """
    current_msg = msg

    for agent in agents:
        current_msg = await agent(current_msg)

    return current_msg
```

**2. fanout_pipeline()**
```python
async def fanout_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None,
    enable_gather: bool = True,
    **kwargs
) -> list[Msg]:
    """并行执行 Agent

    等价于 FanoutPipeline(...)(msg)
    """
    if enable_gather:
        results = await asyncio.gather(*[
            agent(msg, **kwargs)
            for agent in agents
        ])
    else:
        tasks = [
            asyncio.create_task(agent(msg, **kwargs))
            for agent in agents
        ]
        results = tasks

    return results
```

**使用示例**:
```python
# 顺序执行
result = await sequential_pipeline(
    agents=[agent1, agent2, agent3],
    msg=Msg("user", "Start", "user")
)

# 并行执行
results = await fanout_pipeline(
    agents=[agent1, agent2, agent3],
    msg=Msg("user", "Analyze", "user")
)
```

#### 3.7.5 Pipeline 组合模式

**复杂工作流示例**:
```python
async def complex_workflow():
    """复杂的多 Agent 工作流"""

    # 阶段 1: 并行分析
    analysis_results = await fanout_pipeline(
        agents=[analyzer1, analyzer2, analyzer3],
        msg=initial_msg
    )

    # 阶段 2: 汇总分析结果
    summary_agent = SummaryAgent(...)
    summary = await summary_agent(analysis_results)

    # 阶段 3: 多 Agent 讨论
    async with MsgHub(
        participants=[expert1, expert2, expert3],
        announcement=summary
    ) as hub:
        # 顺序发言
        await sequential_pipeline([expert1, expert2, expert3])

    # 阶段 4: 最终决策
    decision_agent = DecisionAgent(...)
    final_decision = await decision_agent()

    return final_decision
```

---

### 3.8 Plan 模块

**文件位置**: `src/agentscope/plan/`

#### 3.8.1 文件结构

```
plan/
├── __init__.py              # 导出接口
├── _plan_model.py          # Plan 和 SubTask 数据模型
├── _plan_notebook.py       # PlanNotebook 计划笔记本
├── _storage_base.py        # PlanStorageBase 存储基类
└── _in_memory_storage.py   # InMemoryPlanStorage 内存存储
```

#### 3.8.2 数据模型

**文件**: `_plan_model.py`

**SubTask 模型**:
```python
class SubTask(BaseModel):
    """子任务模型

    使用 Pydantic 确保数据验证
    """
    name: str                    # 子任务名称
    description: str             # 详细描述
    expected_outcome: str        # 预期结果
    state: Literal["todo", "in_progress", "done", "abandoned"]
    outcome: str | None = None   # 实际结果（完成后填写）

    def to_dict(self) -> dict:
        """序列化为字典"""
        return {
            "name": self.name,
            "description": self.description,
            "expected_outcome": self.expected_outcome,
            "state": self.state,
            "outcome": self.outcome
        }

    @classmethod
    def from_dict(cls, data: dict) -> "SubTask":
        """从字典反序列化"""
        return cls(**data)
```

**Plan 模型**:
```python
class Plan(BaseModel):
    """计划模型"""

    id: str                      # 计划 ID
    name: str                    # 计划名称
    description: str             # 计划描述
    expected_outcome: str        # 预期结果
    subtasks: list[SubTask]      # 子任务列表
    state: Literal["in_progress", "done", "abandoned"]
    created_at: str              # 创建时间
    finished_at: str | None = None  # 完成时间

    def to_dict(self) -> dict:
        """序列化"""
        return {
            "id": self.id,
            "name": self.name,
            "description": self.description,
            "expected_outcome": self.expected_outcome,
            "subtasks": [st.to_dict() for st in self.subtasks],
            "state": self.state,
            "created_at": self.created_at,
            "finished_at": self.finished_at
        }

    @classmethod
    def from_dict(cls, data: dict) -> "Plan":
        """反序列化"""
        data["subtasks"] = [
            SubTask.from_dict(st) for st in data["subtasks"]
        ]
        return cls(**data)
```

#### 3.8.3 PlanNotebook 类

**文件**: `_plan_notebook.py`

PlanNotebook 是计划系统的核心，负责管理当前计划并提供工具函数。

**初始化**:
```python
class PlanNotebook(StateModule):
    """计划笔记本

    特性:
    - 管理当前计划
    - 提供计划相关工具函数
    - 生成计划提示
    - 支持计划状态钩子
    """

    def __init__(
        self,
        storage: PlanStorageBase | None = None,
        enable_hint: bool = True
    ):
        """初始化

        Args:
            storage: 计划存储
            enable_hint: 是否启用计划提示
        """
        super().__init__()
        self.storage = storage or InMemoryPlanStorage()
        self.enable_hint = enable_hint
        self.current_plan: Plan | None = None

        self.register_state("current_plan")
```

**工具函数**:

1. **create_plan() - 创建计划**
```python
async def create_plan(
    self,
    name: str,
    description: str,
    expected_outcome: str,
    subtasks: list[dict]
) -> ToolResponse:
    """创建新计划（工具函数）

    Args:
        name: 计划名称
        description: 计划描述
        expected_outcome: 预期结果
        subtasks: 子任务列表，每个子任务包含:
            - name: 子任务名称
            - description: 描述
            - expected_outcome: 预期结果

    Returns:
        ToolResponse 包含计划 ID
    """
    # 创建子任务对象
    subtask_objs = [
        SubTask(
            name=st["name"],
            description=st["description"],
            expected_outcome=st["expected_outcome"],
            state="todo"
        )
        for st in subtasks
    ]

    # 创建计划
    plan = Plan(
        id=str(uuid.uuid4()),
        name=name,
        description=description,
        expected_outcome=expected_outcome,
        subtasks=subtask_objs,
        state="in_progress",
        created_at=datetime.now().isoformat()
    )

    # 保存计划
    await self.storage.save(plan)
    self.current_plan = plan

    return ToolResponse(
        content=[TextBlock(
            type="text",
            text=f"Plan created with ID: {plan.id}"
        )],
        metadata={"plan_id": plan.id}
    )
```

2. **revise_current_plan() - 修订计划**
```python
async def revise_current_plan(
    self,
    subtask_idx: int,
    action: Literal["add", "modify", "delete"],
    subtask: dict | None = None
) -> ToolResponse:
    """修订当前计划（工具函数）

    Args:
        subtask_idx: 子任务索引（0-based）
        action: 操作类型
            - add: 在指定位置插入新子任务
            - modify: 修改指定子任务
            - delete: 删除指定子任务
        subtask: 子任务数据（add/modify 时需要）

    Returns:
        ToolResponse
    """
    if not self.current_plan:
        return ToolResponse(
            content=[TextBlock(
                type="text",
                text="Error: No current plan"
            )]
        )

    if action == "add":
        # 插入新子任务
        new_subtask = SubTask(
            name=subtask["name"],
            description=subtask["description"],
            expected_outcome=subtask["expected_outcome"],
            state="todo"
        )
        self.current_plan.subtasks.insert(subtask_idx, new_subtask)

    elif action == "modify":
        # 修改子任务
        st = self.current_plan.subtasks[subtask_idx]
        st.name = subtask.get("name", st.name)
        st.description = subtask.get("description", st.description)
        st.expected_outcome = subtask.get(
            "expected_outcome",
            st.expected_outcome
        )

    elif action == "delete":
        # 删除子任务
        del self.current_plan.subtasks[subtask_idx]

    # 保存修改
    await self.storage.save(self.current_plan)

    return ToolResponse(
        content=[TextBlock(
            type="text",
            text=f"Plan revised: {action} at index {subtask_idx}"
        )]
    )
```

3. **update_subtask_state() - 更新子任务状态**
```python
async def update_subtask_state(
    self,
    subtask_idx: int,
    state: Literal["todo", "in_progress", "done", "abandoned"]
) -> ToolResponse:
    """更新子任务状态（工具函数）

    Args:
        subtask_idx: 子任务索引
        state: 新状态
    """
    if not self.current_plan:
        return ToolResponse(
            content=[TextBlock(
                type="text",
                text="Error: No current plan"
            )]
        )

    subtask = self.current_plan.subtasks[subtask_idx]
    subtask.state = state

    await self.storage.save(self.current_plan)

    return ToolResponse(
        content=[TextBlock(
            type="text",
            text=f"Subtask {subtask_idx} state updated to {state}"
        )]
    )
```

4. **finish_subtask() - 完成子任务**
```python
async def finish_subtask(
    self,
    subtask_idx: int,
    subtask_outcome: str
) -> ToolResponse:
    """完成子任务（工具函数）

    Args:
        subtask_idx: 子任务索引
        subtask_outcome: 实际结果
    """
    if not self.current_plan:
        return ToolResponse(
            content=[TextBlock(
                type="text",
                text="Error: No current plan"
            )]
        )

    subtask = self.current_plan.subtasks[subtask_idx]
    subtask.state = "done"
    subtask.outcome = subtask_outcome

    await self.storage.save(self.current_plan)

    return ToolResponse(
        content=[TextBlock(
            type="text",
            text=f"Subtask {subtask_idx} completed"
        )]
    )
```

5. **finish_plan() - 完成计划**
```python
async def finish_plan(
    self,
    state: Literal["done", "abandoned"],
    outcome: str | None = None
) -> ToolResponse:
    """完成或放弃计划（工具函数）

    Args:
        state: 最终状态（done 或 abandoned）
        outcome: 结果说明
    """
    if not self.current_plan:
        return ToolResponse(
            content=[TextBlock(
                type="text",
                text="Error: No current plan"
            )]
        )

    self.current_plan.state = state
    self.current_plan.finished_at = datetime.now().isoformat()

    await self.storage.save(self.current_plan)

    # 清空当前计划
    self.current_plan = None

    return ToolResponse(
        content=[TextBlock(
            type="text",
            text=f"Plan {state}: {outcome or ''}"
        )]
    )
```

**提示生成**:
```python
async def get_current_hint(self) -> Msg | None:
    """根据计划状态生成引导提示

    返回:
        系统消息，包含当前计划状态和下一步建议
    """
    if not self.enable_hint or not self.current_plan:
        return None

    # 查找当前进行中的子任务
    current_subtask = None
    current_idx = None

    for i, subtask in enumerate(self.current_plan.subtasks):
        if subtask.state == "in_progress":
            current_subtask = subtask
            current_idx = i
            break

    # 如果没有进行中的任务，查找下一个待办任务
    if current_subtask is None:
        for i, subtask in enumerate(self.current_plan.subtasks):
            if subtask.state == "todo":
                current_subtask = subtask
                current_idx = i
                break

    if current_subtask is None:
        # 所有子任务都完成了
        hint_text = f"""
Current Plan: {self.current_plan.name}

All subtasks are completed. Please review the results and call finish_plan().
"""
    else:
        # 生成当前子任务提示
        hint_text = f"""
Current Plan: {self.current_plan.name}

Current Subtask ({current_idx + 1}/{len(self.current_plan.subtasks)}):
- Name: {current_subtask.name}
- Description: {current_subtask.description}
- Expected Outcome: {current_subtask.expected_outcome}
- State: {current_subtask.state}

Progress: {self._get_progress_summary()}

Please work on this subtask. When finished, call finish_subtask().
"""

    return Msg(
        name="plan_system",
        content=hint_text,
        role="system"
    )


def _get_progress_summary(self) -> str:
    """生成进度摘要"""
    if not self.current_plan:
        return "No active plan"

    total = len(self.current_plan.subtasks)
    done = sum(1 for st in self.current_plan.subtasks if st.state == "done")
    in_progress = sum(
        1 for st in self.current_plan.subtasks if st.state == "in_progress"
    )

    return f"{done}/{total} done, {in_progress} in progress"
```

**集成到 ReActAgent**:
```python
# 创建计划笔记本
plan_notebook = PlanNotebook()

# 创建 Agent，传入计划笔记本
agent = ReActAgent(
    name="planner_agent",
    model=model,
    memory=InMemoryMemory(),
    formatter=formatter,
    toolkit=toolkit,
    plan_notebook=plan_notebook  # 关键
)

# Agent 会自动:
# 1. 注册计划相关的工具函数
# 2. 在推理时添加计划提示
# 3. 根据计划状态调整行为
```

**使用示例**:
```python
# Agent 创建计划
msg = Msg("user", "Build a web scraper for news websites", "user")
response = await agent(msg)

# Agent 可能会调用 create_plan:
# {
#     "name": "Web Scraper Development",
#     "description": "Build a scraper for news websites",
#     "expected_outcome": "Working scraper with data storage",
#     "subtasks": [
#         {
#             "name": "Research libraries",
#             "description": "Find suitable Python scraping libraries",
#             "expected_outcome": "List of libraries with pros/cons"
#         },
#         {
#             "name": "Design architecture",
#             "description": "Plan scraper structure and data flow",
#             "expected_outcome": "Architecture diagram"
#         },
#         {
#             "name": "Implement scraper",
#             "description": "Code the scraping logic",
#             "expected_outcome": "Working scraper code"
#         },
#         {
#             "name": "Test and debug",
#             "description": "Test with real websites",
#             "expected_outcome": "Bug-free scraper"
#         }
#     ]
#}

# Agent 会按计划执行各个子任务
# 每次调用时，plan_notebook 会提供当前子任务的提示
```

#### 3.8.4 存储接口

**PlanStorageBase**:
```python
class PlanStorageBase(ABC):
    """计划存储基类"""

    @abstractmethod
    async def save(self, plan: Plan) -> None:
        """保存计划"""

    @abstractmethod
    async def load(self, plan_id: str) -> Plan | None:
        """加载计划"""

    @abstractmethod
    async def delete(self, plan_id: str) -> None:
        """删除计划"""

    @abstractmethod
    async def list_all(self) -> list[Plan]:
        """列出所有计划"""
```

**InMemoryPlanStorage**:
```python
class InMemoryPlanStorage(PlanStorageBase):
    """内存计划存储"""

    def __init__(self):
        self.plans: dict[str, Plan] = {}

    async def save(self, plan: Plan) -> None:
        self.plans[plan.id] = plan

    async def load(self, plan_id: str) -> Plan | None:
        return self.plans.get(plan_id)

    async def delete(self, plan_id: str) -> None:
        if plan_id in self.plans:
            del self.plans[plan_id]

    async def list_all(self) -> list[Plan]:
        return list(self.plans.values())
```

---

### 3.9 RAG 模块

**文件位置**: `src/agentscope/rag/`

#### 3.9.1 文件结构

```
rag/
├── __init__.py                # 导出接口
├── _document.py              # Document 文档模型
├── _knowledge_base.py        # KnowledgeBase 知识库基类
├── _simple_knowledge.py      # SimpleKnowledgeBase 简单实现
├── _reader/                  # 文档读取器
│   ├── _reader_base.py
│   ├── _text_reader.py       # 文本文件读取器
│   ├── _pdf_reader.py        # PDF 读取器
│   └── _image_reader.py      # 图片读取器
└── _store/                   # 向量存储
    ├── _store_base.py
    └── _qdrant_store.py      # Qdrant 向量数据库
```

#### 3.9.2 Document 模型

**文件**: `_document.py`

```python
@dataclass
class DocMetadata:
    """文档元数据"""

    content: TextBlock | ImageBlock  # 文档内容
    source: str | None = None        # 来源
    created_at: str | None = None    # 创建时间
    author: str | None = None        # 作者
    tags: list[str] | None = None    # 标签
    # ... 其他元数据字段


class Document:
    """文档类

    包含文档内容、向量嵌入和元数据
    """

    def __init__(
        self,
        metadata: DocMetadata,
        vector: list[float] | None = None,
        id: str | None = None,
        score: float | None = None
    ):
        """初始化文档

        Args:
            metadata: 文档元数据
            vector: 向量嵌入
            id: 文档 ID
            score: 相似度分数（检索时填充）
        """
        self.id = id or str(uuid.uuid4())
        self.metadata = metadata
        self.vector = vector
        self.score = score

    def to_dict(self) -> dict:
        """序列化"""
        return {
            "id": self.id,
            "metadata": asdict(self.metadata),
            "vector": self.vector,
            "score": self.score
        }

    @classmethod
    def from_dict(cls, data: dict) -> "Document":
        """反序列化"""
        return cls(
            id=data["id"],
            metadata=DocMetadata(**data["metadata"]),
            vector=data.get("vector"),
            score=data.get("score")
        )
```

#### 3.9.3 KnowledgeBase 类

**文件**: `_knowledge_base.py`

```python
class KnowledgeBase(StateModule, ABC):
    """知识库基类

    功能:
    - 添加文档
    - 检索相关文档
    - 提供工具函数
    """

    def __init__(
        self,
        name: str,
        embedding_model: EmbeddingModelBase,
        store: VDBStoreBase,
        reader: ReaderBase | None = None
    ):
        """初始化知识库

        Args:
            name: 知识库名称
            embedding_model: 嵌入模型
            store: 向量存储
            reader: 文档读取器
        """
        super().__init__()
        self.name = name
        self.embedding_model = embedding_model
        self.store = store
        self.reader = reader

    @abstractmethod
    async def add_documents(
        self,
        documents: list[Document],
        **kwargs
    ) -> None:
        """添加文档到知识库

        Args:
            documents: 文档列表
        """

    @abstractmethod
    async def retrieve(
        self,
        query: str,
        limit: int = 5,
        score_threshold: float | None = None,
        **kwargs
    ) -> list[Document]:
        """检索相关文档

        Args:
            query: 查询文本
            limit: 返回文档数量
            score_threshold: 相似度阈值

        Returns:
            相关文档列表（按相似度排序）
        """

    async def retrieve_knowledge(
        self,
        query: str,
        limit: int = 5,
        score_threshold: float | None = None
    ) -> ToolResponse:
        """检索工具函数（供 Agent 调用）

        Args:
            query: 查询文本
            limit: 返回数量
            score_threshold: 相似度阈值

        Returns:
            ToolResponse 包含检索到的文档
        """
        docs = await self.retrieve(query, limit, score_threshold)

        # 格式化文档内容
        content_parts = []

        for i, doc in enumerate(docs):
            text = f"Document {i+1} (score: {doc.score:.3f}):\n"

            if doc.metadata.content["type"] == "text":
                text += doc.metadata.content["text"]
            elif doc.metadata.content["type"] == "image":
                text += f"[Image: {doc.metadata.source}]"

            content_parts.append(text)

        content_text = "\n\n".join(content_parts)

        return ToolResponse(
            content=[TextBlock(type="text", text=content_text)],
            metadata={"docs": [doc.to_dict() for doc in docs]}
        )
```

#### 3.9.4 SimpleKnowledgeBase 实现

**文件**: `_simple_knowledge.py`

```python
class SimpleKnowledgeBase(KnowledgeBase):
    """简单知识库实现

    特性:
    - 自动向量化
    - 基于向量相似度检索
    - 支持多种文档格式
    """

    async def add_documents(
        self,
        documents: list[Document],
        **kwargs
    ) -> None:
        """添加文档"""

        # 1. 提取需要向量化的文本
        texts = []
        for doc in documents:
            if doc.metadata.content["type"] == "text":
                texts.append(doc.metadata.content["text"])
            elif doc.metadata.content["type"] == "image":
                # 图片使用多模态嵌入
                texts.append("")  # 占位

        # 2. 批量向量化
        embeddings = await self.embedding_model(texts)

        # 3. 更新文档向量
        for doc, embedding in zip(documents, embeddings):
            doc.vector = embedding.vector

        # 4. 存储到向量数据库
        await self.store.add(documents)

    async def retrieve(
        self,
        query: str,
        limit: int = 5,
        score_threshold: float | None = None,
        **kwargs
    ) -> list[Document]:
        """检索文档"""

        # 1. 向量化查询
        query_embedding = await self.embedding_model([query])
        query_vector = query_embedding[0].vector

        # 2. 向量检索
        docs = await self.store.search(
            vector=query_vector,
            limit=limit,
            score_threshold=score_threshold
        )

        return docs

    async def add_files(
        self,
        file_paths: list[str],
        **kwargs
    ) -> None:
        """从文件添加文档

        Args:
            file_paths: 文件路径列表
        """
        if not self.reader:
            raise ValueError("No reader configured")

        # 读取文件
        documents = await self.reader.read(file_paths)

        # 添加到知识库
        await self.add_documents(documents, **kwargs)
```

**使用示例**:
```python
from agentscope.rag import SimpleKnowledgeBase
from agentscope.rag._reader import TextReader, PDFReader
from agentscope.rag._store import QdrantStore
from agentscope.embedding import DashScopeTextEmbedding

# 1. 创建嵌入模型
embedding_model = DashScopeTextEmbedding(
    model_name="text-embedding-v1",
    api_key="your_api_key"
)

# 2. 创建向量存储
store = QdrantStore(
    collection_name="my_knowledge",
    url="http://localhost:6333"
)

# 3. 创建读取器
reader = TextReader()

# 4. 创建知识库
kb = SimpleKnowledgeBase(
    name="my_kb",
    embedding_model=embedding_model,
    store=store,
    reader=reader
)

# 5. 添加文档
await kb.add_files([
    "docs/manual.txt",
    "docs/faq.txt"
])

# 或手动添加文档
doc = Document(
    metadata=DocMetadata(
        content=TextBlock(type="text", text="Python is a programming language"),
        source="manual",
        tags=["programming", "python"]
    )
)
await kb.add_documents([doc])

# 6. 检索
results = await kb.retrieve(
    query="What is Python?",
    limit=3,
    score_threshold=0.7
)

for doc in results:
    print(f"Score: {doc.score:.3f}")
    print(doc.metadata.content["text"])
    print("---")
```

#### 3.9.5 集成到 ReActAgent

```python
# 创建知识库
kb1 = SimpleKnowledgeBase(...)
kb2 = SimpleKnowledgeBase(...)

# 添加文档
await kb1.add_files(["docs/python_manual.pdf"])
await kb2.add_files(["docs/javascript_manual.pdf"])

# 创建 Agent，传入知识库
agent = ReActAgent(
    name="coding_assistant",
    model=model,
    memory=InMemoryMemory(),
    formatter=formatter,
    toolkit=toolkit,
    knowledge=[kb1, kb2],          # 多个知识库
    enable_rewrite_query=True      # 启用查询重写
)

# Agent 会自动:
# 1. 注册 retrieve_knowledge 工具函数
# 2. 在每次 reply() 开始时，从知识库检索相关信息
# 3. 将检索结果添加到记忆中
# 4. Agent 也可以主动调用检索工具

# 使用
response = await agent(Msg("user", "How do I use list comprehension in Python?", "user"))
# Agent 会自动从 kb1 检索 Python 相关文档，然后生成回复
```

#### 3.9.6 Reader 组件

**TextReader**:
```python
class TextReader(ReaderBase):
    """文本文件读取器"""

    async def read(
        self,
        file_paths: list[str],
        chunk_size: int = 1000,
        chunk_overlap: int = 200
    ) -> list[Document]:
        """读取文本文件

        Args:
            file_paths: 文件路径列表
            chunk_size: 分块大小（字符数）
            chunk_overlap: 分块重叠（字符数）

        Returns:
            文档列表
        """
        documents = []

        for file_path in file_paths:
            with open(file_path, "r", encoding="utf-8") as f:
                text = f.read()

            # 分块
            chunks = self._split_text(text, chunk_size, chunk_overlap)

            # 创建文档
            for i, chunk in enumerate(chunks):
                doc = Document(
                    metadata=DocMetadata(
                        content=TextBlock(type="text", text=chunk),
                        source=file_path,
                        created_at=datetime.now().isoformat(),
                        tags=[f"chunk_{i}"]
                    )
                )
                documents.append(doc)

        return documents

    def _split_text(
        self,
        text: str,
        chunk_size: int,
        chunk_overlap: int
    ) -> list[str]:
        """分割文本"""
        chunks = []
        start = 0

        while start < len(text):
            end = start + chunk_size
            chunk = text[start:end]
            chunks.append(chunk)
            start = end - chunk_overlap

        return chunks
```

**PDFReader**:
```python
class PDFReader(ReaderBase):
    """PDF 文件读取器"""

    async def read(
        self,
        file_paths: list[str],
        **kwargs
    ) -> list[Document]:
        """读取 PDF 文件"""
        from pypdf import PdfReader

        documents = []

        for file_path in file_paths:
            reader = PdfReader(file_path)

            for page_num, page in enumerate(reader.pages):
                text = page.extract_text()

                doc = Document(
                    metadata=DocMetadata(
                        content=TextBlock(type="text", text=text),
                        source=f"{file_path}:page_{page_num+1}",
                        created_at=datetime.now().isoformat()
                    )
                )
                documents.append(doc)

        return documents
```

#### 3.9.7 VDB Store 组件

**QdrantStore**:
```python
class QdrantStore(VDBStoreBase):
    """Qdrant 向量数据库存储"""

    def __init__(
        self,
        collection_name: str,
        url: str = "http://localhost:6333",
        api_key: str | None = None,
        vector_size: int = 1536
    ):
        """初始化

        Args:
            collection_name: 集合名称
            url: Qdrant 服务 URL
            api_key: API Key
            vector_size: 向量维度
        """
        from qdrant_client import QdrantClient
        from qdrant_client.models import Distance, VectorParams

        self.client = QdrantClient(url=url, api_key=api_key)
        self.collection_name = collection_name

        # 创建集合（如果不存在）
        try:
            self.client.create_collection(
                collection_name=collection_name,
                vectors_config=VectorParams(
                    size=vector_size,
                    distance=Distance.COSINE
                )
            )
        except Exception:
            pass  # 集合已存在

    async def add(self, documents: list[Document]) -> None:
        """添加文档"""
        from qdrant_client.models import PointStruct

        points = [
            PointStruct(
                id=doc.id,
                vector=doc.vector,
                payload=doc.to_dict()
            )
            for doc in documents
        ]

        self.client.upsert(
            collection_name=self.collection_name,
            points=points
        )

    async def search(
        self,
        vector: list[float],
        limit: int = 5,
        score_threshold: float | None = None
    ) -> list[Document]:
        """搜索相似文档"""
        results = self.client.search(
            collection_name=self.collection_name,
            query_vector=vector,
            limit=limit,
            score_threshold=score_threshold
        )

        documents = []
        for result in results:
            doc_dict = result.payload
            doc = Document.from_dict(doc_dict)
            doc.score = result.score
            documents.append(doc)

        return documents
```

---

### 3.10 MCP 模块

**文件位置**: `src/agentscope/mcp/`

#### 3.10.1 文件结构

```
mcp/
├── __init__.py                    # 导出接口
├── _client_base.py               # MCPClientBase 基类
├── _mcp_function.py              # MCPToolFunction 包装器
├── _stateful_client_base.py     # 有状态客户端基类
├── _stdio_stateful_client.py    # StdIO 有状态客户端
├── _http_stateless_client.py    # HTTP 无状态客户端
└── _http_stateful_client.py     # HTTP 有状态客户端
```

#### 3.10.2 MCP 协议简介

**Model Context Protocol (MCP)** 是一个标准协议，用于 LLM 应用与外部工具/服务通信。

**核心概念**:
- **Server**: 提供工具/资源的服务
- **Client**: 消费工具/资源的应用
- **Transport**: 通信方式（HTTP, SSE, StdIO）
- **Stateful vs Stateless**: 是否维持会话状态

#### 3.10.3 MCPClientBase 类

**文件**: `_client_base.py`

```python
class MCPClientBase(StateModule, ABC):
    """MCP 客户端基类

    定义所有 MCP 客户端的统一接口
    """

    def __init__(
        self,
        name: str,
        transport: Literal["http", "sse", "stdio"]
    ):
        """初始化

        Args:
            name: 客户端名称
            transport: 传输协议
        """
        super().__init__()
        self.name = name
        self.transport = transport

    @abstractmethod
    async def initialize(self) -> None:
        """初始化连接"""

    @abstractmethod
    async def list_tools(self) -> list[Tool]:
        """列出可用工具"""

    @abstractmethod
    async def call_tool(
        self,
        tool_name: str,
        arguments: dict
    ) -> Any:
        """调用工具"""

    @abstractmethod
    async def get_callable_function(
        self,
        func_name: str
    ) -> Callable:
        """获取可调用函数

        返回一个 Python 函数，可以直接调用或注册到 Toolkit
        """

    @abstractmethod
    async def close(self) -> None:
        """关闭连接"""
```

#### 3.10.4 HttpStatelessClient 类

**文件**: `_http_stateless_client.py`

无状态 HTTP 客户端，每次请求独立。

```python
class HttpStatelessClient(MCPClientBase):
    """HTTP 无状态客户端

    特性:
    - 每次请求独立
    - 无需维护会话
    - 适用于简单的工具调用
    """

    def __init__(
        self,
        name: str,
        url: str,
        headers: dict | None = None,
        timeout: int = 30
    ):
        """初始化

        Args:
            name: 客户端名称
            url: MCP 服务器 URL
            headers: 自定义 HTTP 头
            timeout: 请求超时（秒）
        """
        super().__init__(name=name, transport="http")
        self.url = url
        self.headers = headers or {}
        self.timeout = timeout

    async def initialize(self) -> None:
        """无状态客户端无需初始化"""
        pass

    async def list_tools(self) -> list[Tool]:
        """列出可用工具"""
        import aiohttp

        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.url}/list_tools",
                headers=self.headers,
                timeout=self.timeout
            ) as response:
                response.raise_for_status()
                data = await response.json()

                return [Tool(**tool) for tool in data["tools"]]

    async def call_tool(
        self,
        tool_name: str,
        arguments: dict
    ) -> Any:
        """调用工具"""
        import aiohttp

        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.url}/call_tool",
                json={
                    "tool_name": tool_name,
                    "arguments": arguments
                },
                headers=self.headers,
                timeout=self.timeout
            ) as response:
                response.raise_for_status()
                data = await response.json()

                return data["result"]

    async def get_callable_function(
        self,
        func_name: str
    ) -> Callable:
        """获取可调用函数"""

        # 获取工具 Schema
        tools = await self.list_tools()
        tool = next((t for t in tools if t.name == func_name), None)

        if not tool:
            raise ValueError(f"Tool '{func_name}' not found")

        # 创建 MCPToolFunction
        mcp_func = MCPToolFunction(
            client=self,
            tool_name=func_name,
            tool_schema=tool.to_json_schema()
        )

        return mcp_func

    async def close(self) -> None:
        """无状态客户端无需关闭"""
        pass
```

**使用示例**:
```python
from agentscope.mcp import HttpStatelessClient

# 创建客户端
client = HttpStatelessClient(
    name="gaode_mcp",
    url="https://mcp.amap.com/mcp?key=YOUR_API_KEY"
)

# 列出工具
tools = await client.list_tools()
for tool in tools:
    print(f"Tool: {tool.name} - {tool.description}")

# 调用工具
result = await client.call_tool(
    tool_name="maps_geo",
    arguments={"address": "天安门", "city": "北京"}
)
print(result)

# 获取可调用函数
maps_geo = await client.get_callable_function("maps_geo")

# 直接调用
result = await maps_geo(address="天安门", city="北京")

# 或注册到 Toolkit
toolkit = Toolkit()
await toolkit.register_mcp_client(client)
```

#### 3.10.5 HttpStatefulClient 类

**文件**: `_http_stateful_client.py`

有状态 HTTP 客户端，维护会话状态。

```python
class HttpStatefulClient(MCPClientBase):
    """HTTP 有状态客户端

    特性:
    - 维护会话状态
    - 支持上下文管理
    - 适用于需要会话的场景
    """

    def __init__(
        self,
        name: str,
        url: str,
        headers: dict | None = None,
        timeout: int = 30
    ):
        super().__init__(name=name, transport="http")
        self.url = url
        self.headers = headers or {}
        self.timeout = timeout
        self.session_id: str | None = None

    async def initialize(self) -> None:
        """初始化会话"""
        import aiohttp

        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.url}/initialize",
                headers=self.headers,
                timeout=self.timeout
            ) as response:
                response.raise_for_status()
                data = await response.json()
                self.session_id = data["session_id"]

    async def call_tool(
        self,
        tool_name: str,
        arguments: dict
    ) -> Any:
        """调用工具（带会话 ID）"""
        import aiohttp

        if not self.session_id:
            raise RuntimeError("Client not initialized")

        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.url}/call_tool",
                json={
                    "session_id": self.session_id,
                    "tool_name": tool_name,
                    "arguments": arguments
                },
                headers=self.headers,
                timeout=self.timeout
            ) as response:
                response.raise_for_status()
                data = await response.json()

                return data["result"]

    async def close(self) -> None:
        """关闭会话"""
        import aiohttp

        if not self.session_id:
            return

        async with aiohttp.ClientSession() as session:
            await session.post(
                f"{self.url}/close",
                json={"session_id": self.session_id},
                headers=self.headers,
                timeout=self.timeout
            )

        self.session_id = None

    async def __aenter__(self):
        """上下文管理器入口"""
        await self.initialize()
        return self

    async def __aexit__(self, *args):
        """上下文管理器退出"""
        await self.close()
```

**使用示例**:
```python
# 使用上下文管理器
async with HttpStatefulClient(
    name="my_mcp",
    url="http://localhost:8000/mcp"
) as client:
    # 自动初始化
    tools = await client.list_tools()

    # 调用工具（会话状态保持）
    result1 = await client.call_tool("step1", {})
    result2 = await client.call_tool("step2", {"prev": result1})

    # 退出时自动关闭
```

#### 3.10.6 StdIOStatefulClient 类

**文件**: `_stdio_stateful_client.py`

通过标准输入/输出与 MCP 服务器通信。

```python
class StdIOStatefulClient(MCPClientBase):
    """StdIO 有状态客户端

    特性:
    - 通过 stdin/stdout 通信
    - 适用于本地进程
    - 自动管理子进程
    """

    def __init__(
        self,
        name: str,
        command: str,
        args: list[str] | None = None,
        env: dict | None = None
    ):
        """初始化

        Args:
            name: 客户端名称
            command: 启动 MCP 服务器的命令
            args: 命令参数
            env: 环境变量
        """
        super().__init__(name=name, transport="stdio")
        self.command = command
        self.args = args or []
        self.env = env
        self.process: asyncio.subprocess.Process | None = None

    async def initialize(self) -> None:
        """启动 MCP 服务器进程"""
        import os

        # 合并环境变量
        full_env = os.environ.copy()
        if self.env:
            full_env.update(self.env)

        # 启动进程
        self.process = await asyncio.create_subprocess_exec(
            self.command,
            *self.args,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            env=full_env
        )

        # 等待初始化完成
        await self._wait_ready()

    async def _wait_ready(self) -> None:
        """等待服务器就绪"""
        # 读取就绪信号
        line = await self.process.stdout.readline()
        if b"READY" not in line:
            raise RuntimeError("MCP server failed to start")

    async def call_tool(
        self,
        tool_name: str,
        arguments: dict
    ) -> Any:
        """调用工具"""
        if not self.process:
            raise RuntimeError("Client not initialized")

        # 发送请求
        request = {
            "method": "call_tool",
            "params": {
                "tool_name": tool_name,
                "arguments": arguments
            }
        }

        request_json = json.dumps(request) + "\n"
        self.process.stdin.write(request_json.encode())
        await self.process.stdin.drain()

        # 读取响应
        response_line = await self.process.stdout.readline()
        response = json.loads(response_line.decode())

        if "error" in response:
            raise RuntimeError(response["error"])

        return response["result"]

    async def close(self) -> None:
        """关闭进程"""
        if self.process:
            self.process.terminate()
            await self.process.wait()
            self.process = None
```

**使用示例**:
```python
# 启动本地 MCP 服务器
async with StdIOStatefulClient(
    name="local_mcp",
    command="python",
    args=["mcp_server.py"],
    env={"API_KEY": "xxx"}
) as client:
    # 使用工具
    result = await client.call_tool("analyze", {"text": "..."})
```

#### 3.10.7 MCPToolFunction 类

**文件**: `_mcp_function.py`

将 MCP 工具包装为 Python 函数。

```python
class MCPToolFunction:
    """MCP 工具函数包装器

    特性:
    - 将 MCP 工具转换为可调用函数
    - 自动生成文档字符串
    - 支持类型提示
    - 可直接注册到 Toolkit
    """

    def __init__(
        self,
        client: MCPClientBase,
        tool_name: str,
        tool_schema: dict
    ):
        """初始化

        Args:
            client: MCP 客户端
            tool_name: 工具名称
            tool_schema: 工具 JSON Schema
        """
        self.client = client
        self.tool_name = tool_name
        self.tool_schema = tool_schema

        # 设置函数属性
        self.__name__ = tool_name
        self.__doc__ = self._generate_docstring()

    def _generate_docstring(self) -> str:
        """生成文档字符串"""
        desc = self.tool_schema.get("description", "")
        params = self.tool_schema.get("parameters", {}).get("properties", {})

        doc = f"{desc}\n\nArgs:\n"

        for param_name, param_info in params.items():
            param_desc = param_info.get("description", "")
            doc += f"    {param_name}: {param_desc}\n"

        return doc

    async def __call__(self, **kwargs) -> ToolResponse:
        """调用 MCP 工具"""
        # 调用 MCP
        result = await self.client.call_tool(
            tool_name=self.tool_name,
            arguments=kwargs
        )

        # 转换为 ToolResponse
        if isinstance(result, str):
            content = [TextBlock(type="text", text=result)]
        elif isinstance(result, dict):
            content = [TextBlock(type="text", text=json.dumps(result, indent=2))]
        else:
            content = [TextBlock(type="text", text=str(result))]

        return ToolResponse(
            content=content,
            metadata={"mcp_client": self.client.name}
        )
```

#### 3.10.8 MCP 集成模式

**模式 1: 直接调用**
```python
# 获取函数
func = await client.get_callable_function("my_tool")

# 直接调用
result = await func(arg1="value1", arg2="value2")
```

**模式 2: 注册到 Toolkit**
```python
# 注册整个 MCP 客户端
await toolkit.register_mcp_client(
    client,
    group_name="mcp_tools",
    enable_funcs=["func1", "func2"]  # 可选过滤
)

# 或注册单个函数
func = await client.get_callable_function("my_tool")
toolkit.register_tool_function(func)
```

**模式 3: 集成到 Agent**
```python
# 创建 MCP 客户端
mcp_client = HttpStatelessClient(...)

# 创建 Toolkit 并注册
toolkit = Toolkit()
await toolkit.register_mcp_client(mcp_client)

# 创建 Agent
agent = ReActAgent(
    name="agent",
    model=model,
    memory=memory,
    formatter=formatter,
    toolkit=toolkit  # MCP 工具已包含
)

# Agent 可以直接使用 MCP 工具
await agent(Msg("user", "Use the MCP tool to...", "user"))
```

---

### 3.11 其他核心模块

#### 3.11.1 Embedding 模块

**文件位置**: `src/agentscope/embedding/`

**支持的嵌入模型**:
```python
# DashScope 文本嵌入
embedding_model = DashScopeTextEmbedding(
    model_name="text-embedding-v1",
    api_key="your_api_key"
)

# DashScope 多模态嵌入
multimodal_embedding = DashScopeMultiModalEmbedding(
    model_name="multimodal-embedding-v1",
    api_key="your_api_key"
)

# OpenAI 嵌入
openai_embedding = OpenAITextEmbedding(
    model_name="text-embedding-3-large",
    api_key="your_api_key"
)

# 使用
texts = ["Hello world", "Python programming"]
embeddings = await embedding_model(texts)

for emb in embeddings:
    print(f"Vector dimension: {len(emb.vector)}")
    print(f"Usage: {emb.usage}")
```

**缓存机制**:
```python
from agentscope.embedding import FileEmbeddingCache

# 创建缓存
cache = FileEmbeddingCache(cache_dir="./embeddings_cache")

# 创建带缓存的嵌入模型
embedding_model = DashScopeTextEmbedding(
    model_name="text-embedding-v1",
    api_key="your_api_key",
    cache=cache
)

# 首次调用 -> API 请求
emb1 = await embedding_model(["Hello"])

# 第二次调用相同文本 -> 从缓存读取
emb2 = await embedding_model(["Hello"])
```

#### 3.11.2 Token 模块

**文件位置**: `src/agentscope/token/`

**Token 计数**:
```python
from agentscope.token import OpenAITokenCounter

# 创建计数器
counter = OpenAITokenCounter(model_name="gpt-4")

# 计算消息的 token 数
messages = [
    {"role": "user", "content": "Hello"},
    {"role": "assistant", "content": "Hi there!"}
]

token_count = counter.count(messages)
print(f"Total tokens: {token_count}")

# 估算成本
cost = counter.estimate_cost(token_count)
print(f"Estimated cost: ${cost:.4f}")
```

**支持的计数器**:
- `OpenAITokenCounter`: OpenAI 模型
- `AnthropicTokenCounter`: Anthropic 模型
- `GeminiTokenCounter`: Gemini 模型
- `HuggingFaceTokenCounter`: HuggingFace 模型

#### 3.11.3 Session 模块

**文件位置**: `src/agentscope/session/`

**会话管理**:
```python
from agentscope.session import JsonSession

# 创建会话
session = JsonSession(
    session_dir="./sessions",
    session_id="user_001"
)

# 保存 Agent 状态
agent = ReActAgent(...)
session.save_agent(agent)

# 保存其他数据
session.save("conversation_history", messages)

# 加载会话
loaded_agent = session.load_agent("agent_name")
history = session.load("conversation_history")
```

#### 3.11.4 Evaluate 模块

**文件位置**: `src/agentscope/evaluate/`

**评估框架**:
```python
from agentscope.evaluate import Evaluator, Task, Solution

# 定义任务
task = Task(
    id="task_001",
    description="Solve a math problem",
    inputs={"problem": "What is 2+2?"},
    expected_output="4"
)

# 定义解决方案
solution = Solution(
    agent=my_agent,
    task=task
)

# 创建评估器
evaluator = Evaluator(
    tasks=[task],
    solutions=[solution],
    metrics=["accuracy", "latency"]
)

# 运行评估
results = await evaluator.run()

for result in results:
    print(f"Task: {result.task_id}")
    print(f"Score: {result.score}")
    print(f"Metrics: {result.metrics}")
```

**分布式评估**:
```python
from agentscope.evaluate import RayEvaluator

# 使用 Ray 进行分布式评估
evaluator = RayEvaluator(
    tasks=tasks,
    solutions=solutions,
    num_workers=4  # 并行度
)

results = await evaluator.run()
```

#### 3.11.5 Tracing 模块

**文件位置**: `src/agentscope/tracing/`

**OpenTelemetry 追踪**:
```python
from agentscope.tracing import setup_tracing

# 设置追踪
setup_tracing(
    endpoint="http://localhost:4317",  # OTLP 端点
    service_name="my_agent_app"
)

# 所有 Agent、Model、Tool 调用都会自动追踪

# 集成第三方平台
# Arize Phoenix
setup_tracing(endpoint="https://app.phoenix.arize.com/v1/traces")

# Langfuse
setup_tracing(endpoint="https://cloud.langfuse.com/api/public/ingestion")
```

**自定义追踪**:
```python
from agentscope.tracing import trace

@trace(name="custom_operation")
async def my_operation(data):
    # 操作会被追踪
    result = process(data)
    return result
```

#### 3.11.6 Hooks 模块

**文件位置**: `src/agentscope/hooks/`

**AgentScope Studio 集成**:
```python
import agentscope

# 初始化时连接 Studio
agentscope.init(
    project="my_project",
    name="run_001",
    studio_url="http://localhost:5000"
)

# 自动启用 Studio 钩子:
# - 消息自动发送到 Studio
# - UserAgent 使用 Studio 输入
# - 追踪数据发送到 Studio
```

#### 3.11.7 Exception 模块

**文件位置**: `src/agentscope/exception/`

**异常类型**:
```python
from agentscope.exception import (
    AgentOrientedExceptionBase,
    ToolInterruptedError,
    ToolNotFoundError,
    ToolInvalidArgumentsError
)

# 工具中断异常
try:
    result = await tool_func()
except ToolInterruptedError as e:
    # 用户中断工具执行
    print(f"Tool interrupted: {e}")

# 工具未找到
try:
    await toolkit.call_tool_function(tool_call)
except ToolNotFoundError as e:
    print(f"Tool not found: {e.tool_name}")

# 工具参数无效
try:
    await toolkit.call_tool_function(tool_call)
except ToolInvalidArgumentsError as e:
    print(f"Invalid arguments: {e.details}")
```

#### 3.11.8 Module 模块 - StateModule

**文件位置**: `src/agentscope/module/`

**StateModule 是所有可序列化组件的基类**:

```python
from agentscope.module import StateModule

class MyComponent(StateModule):
    def __init__(self):
        super().__init__()
        self.counter = 0
        self.data = []
        self.nested = SomeOtherStateModule()

        # 注册需要序列化的状态
        self.register_state("counter")
        self.register_state("data")
        self.register_state("nested")  # 嵌套的 StateModule 会递归处理

    def state_dict(self) -> dict:
        """序列化状态"""
        return {
            "counter": self.counter,
            "data": self.data,
            "nested": self.nested.state_dict()
        }

    def load_state_dict(self, state_dict: dict) -> None:
        """加载状态"""
        self.counter = state_dict["counter"]
        self.data = state_dict["data"]
        self.nested.load_state_dict(state_dict["nested"])
```

**自定义序列化**:
```python
class MyComponent(StateModule):
    def __init__(self):
        super().__init__()
        self.complex_obj = ComplexObject()

        # 注册自定义序列化函数
        self.register_state(
            "complex_obj",
            custom_to_json=lambda obj: obj.to_dict(),
            custom_from_json=lambda data: ComplexObject.from_dict(data)
        )
```

---

## 4. 模块间依赖与调用关系

### 4.1 核心依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                        AgentScope 依赖关系                       │
└─────────────────────────────────────────────────────────────────┘

                            AgentBase
                                │
                ┌───────────────┼───────────────┐
                │               │               │
          MemoryBase      ChatModelBase    Toolkit
                │               │               │
        ┌───────┴───────┐      │        ┌──────┴──────┐
  InMemoryMemory   LongTermMemory  │    ToolFunction  MCPClient
        │               │          │        │               │
        │               │     FormatterBase │               │
        │               │          │        │               │
        └───────┬───────┴──────────┴────────┴───────────────┘
                │
         ReActAgentBase
                │
        ┌───────┴────────┐
        │                │
   ReActAgent      UserAgent
        │
   ┌────┴────┬─────────┬──────────┐
   │         │         │          │
PlanNotebook│    KnowledgeBase   │
   │         │         │          │
Plan    Pipeline    RAG      Tracing
```

### 4.2 数据流图

```
┌──────────┐
│   User   │
└────┬─────┘
     │ Input
     ▼
┌──────────────┐
│  UserAgent   │
└────┬─────────┘
     │ Msg
     ▼
┌──────────────────────────────────────────────┐
│              ReActAgent                       │
│  ┌─────────────────────────────────────┐    │
│  │ 1. Memory.add(msg)                  │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │ 2. LongTermMemory.retrieve()        │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │ 3. KnowledgeBase.retrieve()         │    │
│  │    ├── EmbeddingModel               │    │
│  │    └── VDBStore                     │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │ 4. Formatter.format(memory)         │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │ 5. ChatModel(formatted_msgs)        │    │
│  │    └── ChatResponse                 │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │ 6. Toolkit.call_tool_function()     │    │
│  │    ├── ToolFunction                 │    │
│  │    ├── MCPClient                    │    │
│  │    └── ToolResponse                 │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │ 7. Memory.add(response)             │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │ 8. LongTermMemory.record()          │    │
│  └─────────────────────────────────────┘    │
└────┬─────────────────────────────────────────┘
     │ Msg (Response)
     ▼
┌──────────┐
│   User   │
└──────────┘
```

### 4.3 Pipeline 编排流程

```
┌─────────────────────────────────────────────┐
│           MsgHub Pattern                    │
└─────────────────────────────────────────────┘

    ┌─────────┐
    │  MsgHub │
    └────┬────┘
         │ manages
    ┌────┴─────────────────┐
    │                      │
┌───▼───┐  ┌───▼───┐  ┌───▼───┐
│Agent1 │◄─┤Agent2 │◄─┤Agent3 │
└───┬───┘  └───┬───┘  └───┬───┘
    │          │          │
    └──────────┴──────────┘
       Auto Broadcast

┌─────────────────────────────────────────────┐
│       Sequential Pipeline                   │
└─────────────────────────────────────────────┘

Input ──► Agent1 ──► Agent2 ──► Agent3 ──► Output

┌─────────────────────────────────────────────┐
│        Fanout Pipeline                      │
└─────────────────────────────────────────────┘

         ┌──► Agent1 ──┐
Input ───┼──► Agent2 ──┼──► [Output1, Output2, Output3]
         └──► Agent3 ──┘
```

### 4.4 模块交互序列

**典型的 Agent 执行流程**:

```
User ─────► UserAgent ─────► ReActAgent
                               │
                               ├─► Memory.add()
                               │
                               ├─► LongTermMemory.retrieve()
                               │     └─► Mem0 API
                               │
                               ├─► KnowledgeBase.retrieve()
                               │     ├─► EmbeddingModel
                               │     │     └─► DashScope/OpenAI API
                               │     └─► VDBStore
                               │           └─► Qdrant
                               │
                               ├─► PlanNotebook.get_current_hint()
                               │     └─► Plan
                               │
                               ├─► Memory.get_memory()
                               │
                               ├─► Formatter.format()
                               │
                               ├─► ChatModel()
                               │     └─► DashScope/OpenAI/Anthropic API
                               │           └─► ChatResponse
                               │
                               ├─► Toolkit.call_tool_function()
                               │     ├─► ToolFunction
                               │     │     └─► execute_python_code()
                               │     ├─► MCPClient
                               │     │     └─► MCP Server
                               │     └─► ToolResponse
                               │
                               ├─► Memory.add()
                               │
                               ├─► LongTermMemory.record()
                               │
                               └─► Tracing (All above traced)
                                     └─► OpenTelemetry
                                           └─► Studio/Phoenix/Langfuse
```

---

## 5. 关键设计模式

### 5.1 异步优先模式

**设计理念**: 所有 I/O 操作都是异步的。

**优势**:
- 高并发性能
- 非阻塞执行
- 支持并行工具调用

**实现**:
```python
# 所有核心方法都是异步的
async def reply(self, msg: Msg) -> Msg:
    ...

# 支持并行执行
results = await asyncio.gather(
    agent1(msg),
    agent2(msg),
    agent3(msg)
)
```

### 5.2 钩子模式 (Hook Pattern)

**设计理念**: 在关键执行点插入自定义逻辑。

**实现**:
```python
# AgentBase 定义钩子点
HOOK_TYPES = [
    "pre_reply", "post_reply",
    "pre_print", "post_print",
    "pre_observe", "post_observe"
]

# 执行流程
async def reply(self, *args, **kwargs):
    # 执行 pre_reply 钩子
    kwargs = await self._execute_hooks("pre_reply", kwargs)

    # 核心逻辑
    result = await self._reply_impl(*args, **kwargs)

    # 执行 post_reply 钩子
    result = await self._execute_hooks("post_reply", result)

    return result
```

**用途**:
- 日志记录
- 性能监控
- 消息转发到 Studio
- 自定义行为注入

### 5.3 发布-订阅模式 (Pub-Sub)

**设计理念**: Agent 之间通过订阅关系自动广播消息。

**实现**:
```python
class AgentBase:
    def __init__(self):
        self.subscribers: dict[str, list[AgentBase]] = {}

    def set_subscribers(self, group: str, agents: list[AgentBase]):
        self.subscribers[group] = agents

    async def _broadcast_to_subscribers(self, msg: Msg):
        for agents in self.subscribers.values():
            await asyncio.gather(*[
                agent.observe(msg) for agent in agents
            ])
```

**用途**:
- MsgHub 消息中心
- 多 Agent 协作
- 事件驱动架构

### 5.4 策略模式 (Strategy Pattern)

**设计理念**: 通过可替换的组件实现不同策略。

**实现**:
```python
# Formatter 策略
agent = ReActAgent(
    formatter=DashScopeChatFormatter()  # 或 OpenAIChatFormatter
)

# Memory 策略
agent = ReActAgent(
    memory=InMemoryMemory()  # 或自定义 Memory
)

# 长期记忆模式策略
agent = ReActAgent(
    long_term_memory_mode="agent_control"  # 或 "static_control", "both"
)
```

### 5.5 状态管理模式

**设计理念**: 统一的状态序列化/反序列化机制。

**实现**:
```python
class StateModule:
    def register_state(
        self,
        attr_name: str,
        custom_to_json: Callable | None = None,
        custom_from_json: Callable | None = None
    ):
        """注册状态变量"""

    def state_dict(self) -> dict:
        """递归序列化所有注册的状态"""

    def load_state_dict(self, state_dict: dict):
        """递归加载状态"""
```

**优势**:
- 自动追踪 StateModule 属性
- 支持嵌套序列化
- 自定义序列化函数

### 5.6 工厂模式 (Factory Pattern)

**设计理念**: 通过工厂方法创建复杂对象。

**实现**:
```python
# Toolkit 自动创建 RegisteredToolFunction
toolkit.register_tool_function(my_func)
# 内部创建 RegisteredToolFunction 包装器

# MCP Client 创建可调用函数
func = await client.get_callable_function("tool_name")
# 返回 MCPToolFunction 实例
```

### 5.7 装饰器模式 (Decorator Pattern)

**设计理念**: 动态添加功能。

**实现**:
```python
# 追踪装饰器
@trace(name="custom_operation")
async def my_function():
    ...

# 工具函数后处理
toolkit.register_tool_function(
    tool_func,
    postprocess_func=my_postprocessor  # 装饰原始结果
)
```

### 5.8 适配器模式 (Adapter Pattern)

**设计理念**: 将不同接口适配为统一接口。

**实现**:
```python
# Formatter 适配不同 API 格式
class DashScopeChatFormatter:
    async def format(self, msgs: list[Msg]) -> list[dict]:
        # 转换为 DashScope API 格式

class OpenAIChatFormatter:
    async def format(self, msgs: list[Msg]) -> list[dict]:
        # 转换为 OpenAI API 格式

# MCPToolFunction 适配 MCP 为 Python 函数
class MCPToolFunction:
    async def __call__(self, **kwargs) -> ToolResponse:
        # 调用 MCP，返回 ToolResponse
```

---

## 6. 执行流程分析

### 6.1 完整的 ReActAgent 执行流程

```python
# 用户输入
user_input = Msg("user", "帮我分析这个文件", "user")

# 1. 调用 Agent
response = await agent(user_input)

# 内部执行流程:

# 1.1 执行 pre_reply 钩子
# - 日志记录
# - Studio 通知

# 1.2 调用 reply()
#     1.2.1 添加消息到记忆
#     await self.memory.add(user_input)
#
#     1.2.2 从长期记忆检索（如果启用）
#     if self._static_control:
#         retrieved = await self.long_term_memory.retrieve(
#             query="帮我分析这个文件",
#             agent_id=self.agent_id
#         )
#         await self.memory.add(Msg("system", retrieved, "system"))
#
#     1.2.3 从知识库检索（如果有）
#     for kb in self.knowledge:
#         docs = await kb.retrieve("帮我分析这个文件")
#         # 添加到记忆
#
#     1.2.4 获取计划提示（如果有）
#     if self.plan_notebook:
#         hint = await self.plan_notebook.get_current_hint()
#         # 添加到记忆
#
#     1.2.5 ReAct 循环（最多 max_iters 次）
#     for i in range(self.max_iters):
#
#         # 1.2.5.1 推理 (_reasoning)
#         # - 执行 pre_reasoning 钩子
#         # - 获取记忆
#         memory = await self.memory.get_memory()
#         # - 添加系统提示
#         if self.sys_prompt:
#             memory = [Msg("system", self.sys_prompt, "system")] + memory
#         # - 格式化
#         formatted = await self.formatter.format(
#             msgs=memory,
#             tools=self.toolkit.json_schemas
#         )
#         # - 调用模型
#         model_response = await self.model(
#             messages=formatted,
#             tools=self.toolkit.json_schemas
#         )
#         # - 处理流式/非流式响应
#         if isinstance(model_response, AsyncGenerator):
#             async for chunk in model_response:
#                 await self.print(chunk, last=chunk.is_last)
#                 msg_reasoning = chunk
#         else:
#             msg_reasoning = model_response.to_msg(self.name)
#         # - 添加到记忆
#         await self.memory.add(msg_reasoning)
#         # - 执行 post_reasoning 钩子
#
#         # 1.2.5.2 提取工具调用
#         tool_calls = msg_reasoning.get_content_blocks("tool_use")
#
#         if tool_calls:
#             # 1.2.5.3 执行工具 (_acting)
#             # - 执行 pre_acting 钩子
#             # - 并行执行所有工具
#             tool_results = []
#             for tool_call in tool_calls:
#                 async for response in self.toolkit.call_tool_function(tool_call):
#                     # 流式工具响应
#                     if response.stream:
#                         await self.print(response, last=response.is_last)
#                     if response.is_last:
#                         tool_results.append(response)
#             # - 构造工具结果消息
#             tool_result_msgs = [
#                 Msg(
#                     self.name,
#                     [ToolResultBlock(
#                         type="tool_result",
#                         id=tc["id"],
#                         output=tr.content
#                     )],
#                     "assistant"
#                 )
#                 for tc, tr in zip(tool_calls, tool_results)
#             ]
#             # - 添加到记忆
#             await self.memory.add(tool_result_msgs)
#             # - 执行 post_acting 钩子
#         else:
#             # 没有工具调用，说明是最终回复
#             reply_msg = msg_reasoning
#             break
#
#     1.2.6 记录到长期记忆（如果启用）
#     if self._static_control:
#         await self.long_term_memory.record(
#             messages=[user_input, reply_msg],
#             agent_id=self.agent_id
#         )

# 1.3 执行 post_reply 钩子

# 1.4 广播到订阅者
# await self._broadcast_to_subscribers(reply_msg)

# 1.5 打印消息
# await self.print(reply_msg)

# 返回最终回复
return reply_msg
```

### 6.2 多 Agent 协作流程

```python
# 场景：三个 Agent 讨论一个话题

async def multi_agent_discussion():
    # 创建 Agent
    expert1 = ReActAgent(name="Expert1", ...)
    expert2 = ReActAgent(name="Expert2", ...)
    expert3 = ReActAgent(name="Expert3", ...)

    # 使用 MsgHub
    async with MsgHub(
        participants=[expert1, expert2, expert3],
        announcement=Msg("host", "讨论主题：AI 的未来", "system")
    ) as hub:
        # 阶段 1: 每个 Expert 依次发言
        await sequential_pipeline([expert1, expert2, expert3])

        # 内部执行:
        # - expert1 执行 reply()
        #   - 生成回复 msg1
        #   - 自动广播 msg1 到 expert2 和 expert3
        #   - expert2.observe(msg1)
        #   - expert3.observe(msg1)
        #
        # - expert2 执行 reply()
        #   - 记忆中已有 announcement 和 msg1
        #   - 生成回复 msg2
        #   - 自动广播 msg2 到 expert1 和 expert3
        #
        # - expert3 执行 reply()
        #   - 记忆中已有 announcement, msg1, msg2
        #   - 生成回复 msg3
        #   - 自动广播

        # 阶段 2: 动态添加新参与者
        expert4 = ReActAgent(name="Expert4", ...)
        hub.add(expert4)
        # expert4 会接收到之前的所有消息

        # 阶段 3: 并行发言
        responses = await fanout_pipeline([expert1, expert2, expert3, expert4])

        # 阶段 4: 总结
        await hub.broadcast(Msg("host", "感谢各位专家", "system"))

    # 退出上下文后，订阅关系自动清理
```

### 6.3 工具调用流程

```python
# Agent 决定调用工具

# 1. Model 返回工具调用
model_response = ChatResponse(
    content=[
        ToolUseBlock(
            type="tool_use",
            id="call_123",
            name="execute_python_code",
            input={"code": "print(2+2)"}
        )
    ]
)

# 2. Agent 提取工具调用
tool_calls = msg.get_content_blocks("tool_use")
# [ToolUseBlock(...)]

# 3. 执行工具
async for response in toolkit.call_tool_function(tool_calls[0]):
    # 3.1 查找工具
    # registered = toolkit.tool_functions["execute_python_code"]
    #
    # 3.2 检查工具组是否激活
    # if not toolkit.tool_groups[registered.group_name].active:
    #     raise Error
    #
    # 3.3 合并预设参数
    # kwargs = {**registered.preset_kwargs, **tool_call["input"]}
    # kwargs = {"code": "print(2+2)", "timeout": 30}
    #
    # 3.4 执行工具函数
    # result = await registered.func(**kwargs)
    # # execute_python_code("print(2+2)", timeout=30)
    # # 输出: "4\n"
    #
    # 3.5 包装为 ToolResponse
    # if not isinstance(result, ToolResponse):
    #     result = ToolResponse(
    #         content=[TextBlock(type="text", text=result)]
    #     )
    #
    # 3.6 应用后处理（如果有）
    # if registered.postprocess_func:
    #     result = registered.postprocess_func(result)
    #
    # 3.7 yield 结果
    yield response  # ToolResponse(content=[TextBlock("4\n")])

# 4. Agent 构造工具结果消息
tool_result_msg = Msg(
    "assistant",
    [ToolResultBlock(
        type="tool_result",
        id="call_123",
        output=[TextBlock(type="text", text="4\n")]
    )],
    "assistant"
)

# 5. 添加到记忆
await memory.add(tool_result_msg)

# 6. 下一轮推理
# Model 会看到工具结果，生成最终回复
```

### 6.4 RAG 检索流程

```python
# Agent 需要从知识库检索信息

# 1. Agent 接收查询
query = "Python 列表推导式怎么用？"

# 2. 从知识库检索（在 reply() 开始时自动执行）
for kb in agent.knowledge:
    # 2.1 查询重写（如果启用）
    if agent.enable_rewrite_query:
        rewritten_query = await agent.model([
            Msg("system", "Rewrite this query for better retrieval", "system"),
            Msg("user", query, "user")
        ])
        query = rewritten_query.get_text_content()

    # 2.2 向量化查询
    query_embedding = await kb.embedding_model([query])
    query_vector = query_embedding[0].vector

    # 2.3 向量检索
    docs = await kb.store.search(
        vector=query_vector,
        limit=5,
        score_threshold=0.7
    )
    # 返回: [Document(score=0.92, ...), Document(score=0.85, ...), ...]

    # 2.4 格式化文档
    retrieved_text = "\n\n".join([
        f"文档 {i+1} (相似度: {doc.score:.2f}):\n{doc.metadata.content['text']}"
        for i, doc in enumerate(docs)
    ])

    # 2.5 添加到记忆
    await agent.memory.add(Msg(
        "system",
        f"来自知识库 '{kb.name}' 的相关信息:\n{retrieved_text}",
        "system"
    ))

# 3. 继续正常的推理流程
# Model 会看到检索到的文档，生成更准确的回复
```

---

## 7. 扩展开发指南

### 7.1 自定义 Agent

#### 7.1.1 继承 AgentBase

**基础自定义 Agent**:
```python
from agentscope.agent import AgentBase
from agentscope.message import Msg

class MySimpleAgent(AgentBase):
    """自定义简单 Agent"""

    def __init__(self, name: str, custom_param: str):
        super().__init__(name=name)
        self.custom_param = custom_param

        # 注册状态（使其可序列化）
        self.register_state("custom_param")

    async def reply(self, msg: Msg | None = None) -> Msg:
        """实现核心逻辑"""
        # 自定义处理逻辑
        response_text = f"Received: {msg.get_text_content()}"
        response_text += f"\nCustom param: {self.custom_param}"

        return Msg(
            name=self.name,
            content=response_text,
            role="assistant"
        )
```

#### 7.1.2 继承 ReActAgent

**扩展 ReActAgent**:
```python
from agentscope.agent import ReActAgent

class MyEnhancedAgent(ReActAgent):
    """增强的 ReAct Agent"""

    def __init__(self, *args, custom_feature: bool = True, **kwargs):
        super().__init__(*args, **kwargs)
        self.custom_feature = custom_feature
        self.register_state("custom_feature")

    async def _reasoning(self, *args, **kwargs):
        """重写推理过程"""
        # 添加自定义逻辑
        if self.custom_feature:
            # 自定义预处理
            pass

        # 调用父类推理
        result = await super()._reasoning(*args, **kwargs)

        # 自定义后处理
        return result

    async def _acting(self, tool_calls):
        """重写执行过程"""
        # 添加自定义工具执行逻辑
        results = []

        for tool_call in tool_calls:
            # 自定义工具调用前处理
            print(f"Calling tool: {tool_call['name']}")

            # 执行工具
            async for response in self.toolkit.call_tool_function(tool_call):
                results.append(response)

        return results
```

### 7.2 自定义工具函数

#### 7.2.1 简单工具函数

```python
from agentscope.tool import ToolResponse
from agentscope.message import TextBlock

def my_calculator(
    operation: str,
    a: float,
    b: float
) -> ToolResponse:
    """简单计算器工具

    Args:
        operation: 操作类型 (+, -, *, /)
        a: 第一个数字
        b: 第二个数字
    """
    operations = {
        "+": a + b,
        "-": a - b,
        "*": a * b,
        "/": a / b if b != 0 else "Error: Division by zero"
    }

    result = operations.get(operation, "Unknown operation")

    return ToolResponse(
        content=[TextBlock(type="text", text=str(result))]
    )

# 注册工具
toolkit.register_tool_function(my_calculator)
```

#### 7.2.2 异步工具函数

```python
import aiohttp

async def fetch_webpage(url: str) -> ToolResponse:
    """获取网页内容（异步）

    Args:
        url: 网页 URL
    """
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            text = await response.text()

    return ToolResponse(
        content=[TextBlock(type="text", text=text[:1000])]  # 截取前1000字符
    )

# 注册异步工具
toolkit.register_tool_function(fetch_webpage, group_name="web")
```

#### 7.2.3 流式工具函数

```python
from typing import AsyncGenerator

async def streaming_process(
    data: str
) -> AsyncGenerator[ToolResponse, None]:
    """流式处理工具

    Args:
        data: 输入数据
    """
    # 初始化
    yield ToolResponse(
        content=[TextBlock(type="text", text="Processing started...")],
        stream=True,
        is_last=False
    )

    # 处理过程
    chunks = data.split()
    for i, chunk in enumerate(chunks):
        # 模拟处理
        await asyncio.sleep(0.1)

        is_last = (i == len(chunks) - 1)
        yield ToolResponse(
            content=[TextBlock(type="text", text=f"Processed: {chunk}")],
            stream=True,
            is_last=is_last
        )

# 注册流式工具
toolkit.register_tool_function(streaming_process)
```

### 7.3 自定义 Formatter

```python
from agentscope.formatter import FormatterBase
from agentscope.message import Msg

class MyCustomFormatter(FormatterBase):
    """自定义格式化器"""

    async def format(
        self,
        msgs: list[Msg],
        tools: list[dict] | None = None
    ) -> list[dict]:
        """格式化为自定义 API 格式"""
        self.assert_list_of_msgs(msgs)

        formatted = []

        for msg in msgs:
            # 自定义格式
            formatted_msg = {
                "from": msg.name,
                "message": msg.get_text_content(),
                "type": msg.role
            }

            # 处理工具调用
            tool_uses = msg.get_content_blocks("tool_use")
            if tool_uses:
                formatted_msg["tools"] = [
                    {
                        "name": tu["name"],
                        "args": tu["input"]
                    }
                    for tu in tool_uses
                ]

            formatted.append(formatted_msg)

        return formatted
```

### 7.4 自定义 Memory

```python
from agentscope.memory import MemoryBase
from agentscope.message import Msg
import sqlite3

class SQLiteMemory(MemoryBase):
    """基于 SQLite 的持久化记忆"""

    def __init__(self, db_path: str = "memory.db"):
        super().__init__()
        self.db_path = db_path
        self._init_db()

    def _init_db(self):
        """初始化数据库"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS messages (
                id TEXT PRIMARY KEY,
                name TEXT,
                content TEXT,
                role TEXT,
                timestamp TEXT
            )
        """)
        conn.commit()
        conn.close()

    async def add(
        self,
        memories: Msg | list[Msg] | None,
        allow_duplicates: bool = False
    ) -> None:
        """添加到数据库"""
        if memories is None:
            return

        if not isinstance(memories, list):
            memories = [memories]

        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        for msg in memories:
            msg_dict = msg.to_dict()
            cursor.execute("""
                INSERT OR REPLACE INTO messages
                VALUES (?, ?, ?, ?, ?)
            """, (
                msg_dict["id"],
                msg_dict["name"],
                str(msg_dict["content"]),
                msg_dict["role"],
                msg_dict["timestamp"]
            ))

        conn.commit()
        conn.close()

    async def get_memory(self) -> list[Msg]:
        """从数据库检索"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM messages ORDER BY timestamp")
        rows = cursor.fetchall()
        conn.close()

        # 转换回 Msg 对象
        messages = []
        for row in rows:
            msg = Msg(
                name=row[1],
                content=eval(row[2]),  # 简化处理
                role=row[3]
            )
            msg.id = row[0]
            msg.timestamp = row[4]
            messages.append(msg)

        return messages

    async def clear(self) -> None:
        """清空数据库"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM messages")
        conn.commit()
        conn.close()

    # 实现其他抽象方法...
```

### 7.5 自定义知识库

```python
from agentscope.rag import KnowledgeBase, Document
from agentscope.embedding import EmbeddingModelBase
import numpy as np

class SimpleVectorKnowledgeBase(KnowledgeBase):
    """简单的向量知识库（内存实现）"""

    def __init__(
        self,
        name: str,
        embedding_model: EmbeddingModelBase
    ):
        super().__init__(
            name=name,
            embedding_model=embedding_model,
            store=None,  # 不使用外部存储
            reader=None
        )
        self.documents: list[Document] = []
        self.vectors: np.ndarray | None = None

    async def add_documents(
        self,
        documents: list[Document],
        **kwargs
    ) -> None:
        """添加文档"""
        # 提取文本
        texts = [
            doc.metadata.content["text"]
            for doc in documents
            if doc.metadata.content["type"] == "text"
        ]

        # 向量化
        embeddings = await self.embedding_model(texts)

        # 添加向量
        for doc, emb in zip(documents, embeddings):
            doc.vector = emb.vector
            self.documents.append(doc)

        # 更新向量矩阵
        self._update_vector_matrix()

    def _update_vector_matrix(self):
        """更新向量矩阵"""
        if self.documents:
            self.vectors = np.array([
                doc.vector for doc in self.documents
            ])

    async def retrieve(
        self,
        query: str,
        limit: int = 5,
        score_threshold: float | None = None,
        **kwargs
    ) -> list[Document]:
        """检索相关文档"""
        if not self.documents:
            return []

        # 向量化查询
        query_emb = await self.embedding_model([query])
        query_vector = np.array(query_emb[0].vector)

        # 计算余弦相似度
        similarities = np.dot(self.vectors, query_vector) / (
            np.linalg.norm(self.vectors, axis=1) * np.linalg.norm(query_vector)
        )

        # 排序
        indices = np.argsort(similarities)[::-1]

        # 过滤和限制
        results = []
        for idx in indices:
            score = float(similarities[idx])

            if score_threshold and score < score_threshold:
                continue

            doc = self.documents[idx]
            doc.score = score
            results.append(doc)

            if len(results) >= limit:
                break

        return results
```

### 7.6 自定义 Pipeline

```python
from agentscope.agent import AgentBase
from agentscope.message import Msg

class ConditionalPipeline:
    """条件分支 Pipeline"""

    def __init__(
        self,
        condition_agent: AgentBase,
        true_pipeline: list[AgentBase],
        false_pipeline: list[AgentBase]
    ):
        self.condition_agent = condition_agent
        self.true_pipeline = true_pipeline
        self.false_pipeline = false_pipeline

    async def __call__(self, msg: Msg) -> Msg:
        """执行条件分支"""
        # 获取条件判断
        condition_result = await self.condition_agent(msg)
        condition_text = condition_result.get_text_content().lower()

        # 根据条件选择 Pipeline
        if "yes" in condition_text or "true" in condition_text:
            pipeline = self.true_pipeline
        else:
            pipeline = self.false_pipeline

        # 执行选中的 Pipeline
        current_msg = msg
        for agent in pipeline:
            current_msg = await agent(current_msg)

        return current_msg
```

---

## 8. 最佳实践

### 8.1 Agent 设计最佳实践

#### 8.1.1 明确的系统提示

```python
agent = ReActAgent(
    name="code_reviewer",
    sys_prompt="""你是一个代码审查专家。请遵循以下原则：
    1. 关注代码质量、安全性和性能
    2. 提供具体的改进建议
    3. 使用友好的语气
    4. 优先考虑重要问题
    """,
    model=model,
    ...
)
```

#### 8.1.2 合理的最大迭代次数

```python
# 简单任务
agent = ReActAgent(max_iters=5, ...)

# 复杂任务
agent = ReActAgent(max_iters=20, ...)
```

#### 8.1.3 使用钩子进行监控

```python
def logging_hook(self, kwargs):
    """记录 Agent 调用"""
    print(f"[{datetime.now()}] Agent {self.name} called")
    return kwargs

ReActAgent.register_class_hook("pre_reply", "logging", logging_hook)
```

### 8.2 工具管理最佳实践

#### 8.2.1 工具分组

```python
toolkit = Toolkit()

# 基础工具组（始终激活）
toolkit.register_tool_function(calculate, group_name="basic")

# 文件操作组
toolkit.create_tool_group("file", "File operations", active=False)
toolkit.register_tool_function(read_file, group_name="file")
toolkit.register_tool_function(write_file, group_name="file")

# 网络工具组
toolkit.create_tool_group("web", "Web tools", active=False)
toolkit.register_tool_function(fetch_url, group_name="web")
toolkit.register_tool_function(search, group_name="web")

# 根据需要激活
toolkit.update_tool_groups(["file"], active=True)
```

#### 8.2.2 工具函数文档

```python
def my_tool(param1: str, param2: int = 10) -> ToolResponse:
    """简洁的一句话描述

    更详细的说明（可选）。
    解释工具的用途、适用场景等。

    Args:
        param1: 参数1的清晰描述
        param2: 参数2的清晰描述（默认值：10）

    Returns:
        ToolResponse 包含处理结果
    """
    # 实现
```

#### 8.2.3 错误处理

```python
def robust_tool(param: str) -> ToolResponse:
    """健壮的工具函数"""
    try:
        # 执行操作
        result = risky_operation(param)

        return ToolResponse(
            content=[TextBlock(type="text", text=result)]
        )

    except ValueError as e:
        return ToolResponse(
            content=[TextBlock(
                type="text",
                text=f"参数错误: {str(e)}"
            )],
            metadata={"error": "ValueError"}
        )

    except Exception as e:
        return ToolResponse(
            content=[TextBlock(
                type="text",
                text=f"执行失败: {str(e)}"
            )],
            metadata={"error": type(e).__name__}
        )
```

### 8.3 记忆管理最佳实践

#### 8.3.1 定期清理记忆

```python
# 在长对话中定期清理
async def periodic_cleanup(agent, max_messages=100):
    """定期清理记忆"""
    if await agent.memory.size() > max_messages:
        messages = await agent.memory.get_memory()
        # 保留最近的消息
        recent_messages = messages[-max_messages:]
        await agent.memory.clear()
        await agent.memory.add(recent_messages)
```

#### 8.3.2 长期记忆策略选择

```python
# 场景 1: Agent 完全自主
agent = ReActAgent(
    long_term_memory=ltm,
    long_term_memory_mode="agent_control"  # Agent 决定何时检索/记录
)

# 场景 2: 自动记录每次对话
agent = ReActAgent(
    long_term_memory=ltm,
    long_term_memory_mode="static_control"  # 每次自动检索和记录
)

# 场景 3: 两者结合
agent = ReActAgent(
    long_term_memory=ltm,
    long_term_memory_mode="both"  # 自动 + Agent 可主动调用
)
```

### 8.4 RAG 最佳实践

#### 8.4.1 文档分块策略

```python
# 小块：适合精确匹配
reader = TextReader(chunk_size=500, chunk_overlap=50)

# 大块：适合保留上下文
reader = TextReader(chunk_size=2000, chunk_overlap=200)
```

#### 8.4.2 多知识库组合

```python
# 创建专门的知识库
python_kb = SimpleKnowledgeBase(name="python_docs", ...)
javascript_kb = SimpleKnowledgeBase(name="js_docs", ...)
general_kb = SimpleKnowledgeBase(name="general", ...)

# Agent 同时使用多个知识库
agent = ReActAgent(
    knowledge=[python_kb, javascript_kb, general_kb],
    enable_rewrite_query=True  # 自动优化查询
)
```

#### 8.4.3 检索参数调优

```python
# 严格检索（高精度）
docs = await kb.retrieve(
    query="specific query",
    limit=3,
    score_threshold=0.8  # 高阈值
)

# 宽松检索（高召回）
docs = await kb.retrieve(
    query="broad query",
    limit=10,
    score_threshold=0.5  # 低阈值
)
```

### 8.5 Pipeline 最佳实践

#### 8.5.1 清晰的工作流设计

```python
async def analysis_workflow(data):
    """清晰的多阶段工作流"""

    # 阶段 1: 数据预处理
    preprocessor = ReActAgent(name="preprocessor", ...)
    preprocessed = await preprocessor(data)

    # 阶段 2: 并行分析
    analyzers = [
        ReActAgent(name="analyzer1", ...),
        ReActAgent(name="analyzer2", ...),
        ReActAgent(name="analyzer3", ...)
    ]
    analyses = await fanout_pipeline(analyzers, preprocessed)

    # 阶段 3: 结果汇总
    summarizer = ReActAgent(name="summarizer", ...)
    summary = await summarizer(analyses)

    # 阶段 4: 质量检查
    reviewer = ReActAgent(name="reviewer", ...)
    final_result = await reviewer(summary)

    return final_result
```

#### 8.5.2 使用 MsgHub 进行讨论

```python
async def team_discussion(topic):
    """团队讨论模式"""

    experts = [
        ReActAgent(name="Expert1", sys_prompt="你是...专家", ...),
        ReActAgent(name="Expert2", sys_prompt="你是...专家", ...),
        ReActAgent(name="Expert3", sys_prompt="你是...专家", ...),
    ]

    async with MsgHub(
        participants=experts,
        announcement=Msg("moderator", f"讨论主题：{topic}", "system")
    ) as hub:
        # 第一轮：各自发表观点
        round1 = await sequential_pipeline(experts)

        # 第二轮：基于他人观点进行讨论
        round2 = await sequential_pipeline(experts)

        # 第三轮：达成共识
        round3 = await sequential_pipeline(experts)

    return round3
```

### 8.6 性能优化最佳实践

#### 8.6.1 并行工具调用

```python
# ReActAgent 默认支持并行工具调用
agent = ReActAgent(...)

# Model 可以返回多个工具调用
# Agent 会自动并行执行
```

#### 8.6.2 使用嵌入缓存

```python
from agentscope.embedding import FileEmbeddingCache

cache = FileEmbeddingCache(cache_dir="./cache")

embedding_model = DashScopeTextEmbedding(
    model_name="text-embedding-v1",
    cache=cache  # 启用缓存
)

# 重复的文本会从缓存读取，节省 API 调用
```

#### 8.6.3 流式输出

```python
# 使用流式模型
model = DashScopeChatModel(
    model_name="qwen-max",
    stream=True  # 启用流式输出
)

agent = ReActAgent(model=model, ...)

# 流式响应会实时显示
```

### 8.7 错误处理最佳实践

#### 8.7.1 捕获特定异常

```python
from agentscope.exception import (
    ToolNotFoundError,
    ToolInvalidArgumentsError
)

try:
    result = await agent(msg)
except ToolNotFoundError as e:
    print(f"工具 '{e.tool_name}' 不存在")
    # 处理：注册缺失的工具或使用其他工具

except ToolInvalidArgumentsError as e:
    print(f"工具参数无效: {e.details}")
    # 处理：修正参数或提示用户
```

#### 8.7.2 优雅降级

```python
async def robust_agent_call(agent, msg, max_retries=3):
    """带重试的 Agent 调用"""
    for attempt in range(max_retries):
        try:
            return await agent(msg)
        except Exception as e:
            if attempt == max_retries - 1:
                # 最后一次尝试失败，返回错误消息
                return Msg(
                    "system",
                    f"处理失败: {str(e)}",
                    "system"
                )
            else:
                # 重试
                await asyncio.sleep(2 ** attempt)  # 指数退避
```

### 8.8 测试最佳实践

#### 8.8.1 模拟 Model

```python
class MockModel(ChatModelBase):
    """用于测试的模拟模型"""

    def __init__(self, responses: list[str]):
        self.responses = responses
        self.call_count = 0

    async def __call__(self, messages, **kwargs):
        response = self.responses[self.call_count]
        self.call_count += 1

        return ChatResponse(
            content=[TextBlock(type="text", text=response)],
            id="test",
            created_at=datetime.now().isoformat(),
            type="chat"
        )

# 测试中使用
mock_model = MockModel(responses=["Response 1", "Response 2"])
agent = ReActAgent(model=mock_model, ...)
```

#### 8.8.2 状态序列化测试

```python
async def test_agent_serialization():
    """测试 Agent 状态保存和加载"""
    # 创建 Agent
    agent = ReActAgent(...)

    # 执行一些操作
    await agent(Msg("user", "test", "user"))

    # 保存状态
    state = agent.state_dict()

    # 创建新 Agent 并加载状态
    new_agent = ReActAgent(...)
    new_agent.load_state_dict(state)

    # 验证状态一致
    assert agent.memory.content == new_agent.memory.content
```

---

## 9. 项目总结

### 9.1 核心优势

1. **透明性优先**
   - 所有操作对开发者可见可控
   - 无深度封装，易于理解和调试

2. **异步优先**
   - 全面支持异步执行
   - 高并发性能
   - 支持并行工具调用和 Agent 执行

3. **高度模块化**
   - LEGO 式组件设计
   - 易于扩展和定制
   - 清晰的模块边界

4. **多 Agent 导向**
   - 显式消息传递
   - 灵活的工作流编排
   - MsgHub 发布-订阅模式

5. **企业级特性**
   - 完整的状态管理
   - OpenTelemetry 追踪集成
   - 长期记忆支持
   - RAG 和 MCP 集成

### 9.2 适用场景

#### 9.2.1 单 Agent 应用
- 客服机器人
- 代码助手
- 内容生成
- 数据分析

#### 9.2.2 多 Agent 应用
- 团队协作模拟
- 多角色讨论
- 复杂决策系统
- 游戏 AI（如狼人杀）

#### 9.2.3 企业应用
- 智能客服系统
- 知识管理平台
- 自动化工作流
- 研究助手

### 9.3 技术栈总结

**核心技术**:
- Python 3.10+
- AsyncIO 异步编程
- Pydantic 数据验证
- OpenTelemetry 追踪

**支持的 LLM**:
- 阿里云通义千问 (DashScope)
- OpenAI GPT 系列
- Anthropic Claude 系列
- Google Gemini
- Ollama 本地模型

**集成能力**:
- MCP (Model Context Protocol)
- Mem0 长期记忆
- Qdrant 向量数据库
- AgentScope Studio

### 9.4 项目规模

- **总代码文件**: 143+ Python 文件
- **核心模块**: 18 个主要模块
- **代码量**: 15,000+ 行生产代码
- **版本**: 1.0.4 (稳定版本)

### 9.5 学习路径建议

**初级**:
1. 理解 Msg 和 Agent 基本概念
2. 使用 ReActAgent 创建简单应用
3. 学习基本工具注册和调用

**中级**:
1. 掌握 Pipeline 编排
2. 使用 MsgHub 进行多 Agent 协作
3. 集成 RAG 知识库
4. 使用 Plan 模块进行任务分解

**高级**:
1. 自定义 Agent 和工具
2. 实现自定义 Memory 和 Formatter
3. 集成 MCP 和外部服务
4. 使用 Tracing 进行性能分析
5. 构建企业级应用

### 9.6 参考资源

**官方资源**:
- GitHub: https://github.com/agentscope-ai/agentscope
- 文档: https://doc.agentscope.io/
- Studio: https://github.com/agentscope-ai/agentscope-studio
- Runtime: https://github.com/agentscope-ai/agentscope-runtime

**论文**:
- AgentScope 1.0: https://arxiv.org/abs/2508.16279
- AgentScope: https://arxiv.org/abs/2402.14034

**社区**:
- Discord: https://discord.gg/eYMpfnkG8h
- DingTalk: 扫描 README 中的二维码

### 9.7 未来展望

**已规划功能**:
- 更多模型支持
- 增强的 RAG 能力
- 更丰富的内置 Agent
- 更好的可视化工具
- 性能优化

**可能的扩展方向**:
- 支持更多向量数据库
- 集成更多第三方工具
- 提供更多评估基准
- 多语言 SDK

---

## 附录：快速参考

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

### A.2 常用模式

**基本 Agent**:
```python
agent = ReActAgent(
    name="assistant",
    sys_prompt="You are a helpful assistant",
    model=DashScopeChatModel(...),
    memory=InMemoryMemory(),
    formatter=DashScopeChatFormatter(),
    toolkit=Toolkit()
)

msg = Msg("user", "Hello", "user")
response = await agent(msg)
```

**多 Agent 协作**:
```python
async with MsgHub(participants=[agent1, agent2, agent3]) as hub:
    await sequential_pipeline([agent1, agent2, agent3])
```

**工具注册**:
```python
toolkit.register_tool_function(my_tool, group_name="custom")
```

**知识库集成**:
```python
agent = ReActAgent(
    knowledge=[kb1, kb2],
    enable_rewrite_query=True,
    ...
)
```

---

## 文档完成

本文档详细分析了 AgentScope 项目的：
- ✅ 项目结构和组织
- ✅ 18 个核心模块的实现
- ✅ 模块间依赖和调用关系
- ✅ 关键设计模式
- ✅ 完整执行流程
- ✅ 扩展开发指南
- ✅ 最佳实践建议

**文档总字数**: 约 50,000+ 字
**代码示例**: 200+ 个
**覆盖模块**: 18 个核心模块

希望这份文档能帮助您深入理解和使用 AgentScope 框架！

---

*最后更新: 2025-10-07*
*文档版本: 2.0*