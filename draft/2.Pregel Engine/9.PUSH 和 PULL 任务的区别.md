# PUSH 和 PULL 任务的区别

## 概述

`prepare_next_tasks()` 函数是 LangGraph Pregel 执行引擎的核心函数，负责准备下一个 superstep（超级步骤）中需要执行的任务。它返回两类任务的集合：

1. **PUSH 任务**：处理之前步骤中排队的 Send 操作
2. **PULL 任务**：确定哪些节点应该因为输入通道更新而被触发执行

## 核心区别对比

| 特性 | PUSH 任务 | PULL 任务 |
|------|-----------|-----------|
| **触发方式** | 显式 Send 操作 | 通道更新自动触发 |
| **任务来源** | TASKS 通道中的 Send 对象 | 通道版本变化 |
| **输入来源** | Send 对象的 `arg` 字段 | 从节点的 `channels`（输入通道）读取 |
| **触发机制** | 基于 TASKS 通道中的 Send 对象 | 基于 `triggers`（触发通道）的版本变化 |
| **任务路径** | `(PUSH, idx)` 或 `(PUSH, ...)` | `(PULL, node_name)` |
| **使用场景** | 动态调用、Map-Reduce、并行处理 | 标准边、条件边、工具调用 |
| **灵活性** | 可传递任意数据，不受节点配置限制 | 受节点配置限制，使用统一状态结构 |
| **调用数量** | 动态，运行时决定 | 固定或基于条件 |

## PUSH 任务（推送任务）

### 定义

PUSH 任务代表在**上一个 superstep** 中由节点或边排队的 `Send` 操作。这些任务在下一个 superstep 中执行，用于将消息传递给目标节点。

### 工作原理

1. **任务来源**：来自 `TASKS` 通道（一个 `Topic[Send]` 类型的通道）
2. **创建时机**：当节点或边需要动态调用另一个节点时，会创建一个 `Send` 对象并写入 `TASKS` 通道
3. **输入处理**：PUSH 任务**忽略**节点的 `channels` 属性，直接从 `Send.arg` 获取输入数据
4. **执行**：在下一个 superstep 中，每个 Send 都会成为一个 PUSH 任务

### 使用场景

**Map-Reduce 模式**：
```python
def continue_to_jokes(state: OverallState):
    # 为每个 subject 创建一个 Send 操作
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

# 这些 Send 操作会被写入 TASKS 通道
# 在下一个 superstep 中，每个 Send 都会成为一个 PUSH 任务
```

**并行处理列表**：
```python
def continue_to_weather(state: OverallState) -> list[Send]:
    # 为每个 location 创建一个 Send
    return [
        Send("get_weather", {"location": location})
        for location in state["locations"]
    ]
```

### 特点

- ✅ **动态数量**：运行时才知道要调用多少次
- ✅ **不同输入**：每次调用可以传递不同的输入数据
- ✅ **并行处理**：天然支持并行执行
- ✅ **灵活控制**：可以传递与当前状态无关的数据

## PULL 任务（拉取任务）

### 定义

PULL 任务代表应该执行的节点，因为它们的**输入通道已被更新**。这些任务基于通道版本变化来触发节点执行。

### 工作原理

1. **触发检查**：通过 `_triggers()` 函数检查节点的 `triggers`（触发通道）是否更新
2. **输入准备**：从节点的 `channels`（输入通道）读取数据作为输入
3. **优化策略**：使用 `updated_channels` 和 `trigger_to_nodes` 快速定位需要执行的节点
4. **执行**：在下一个 superstep 中，被触发的节点会作为 PULL 任务执行

### 使用场景

**标准边触发**：
```python
# 当通道 "messages" 更新时，触发 "process_message" 节点
builder.add_edge("input", "process_message")
# 在下一个 superstep 中，会创建一个 PULL 任务来执行 "process_message"
```

**条件边触发**：
```python
# 根据条件决定下一个节点
def route(state):
    if state["status"] == "ready":
        return "process"
    return "wait"

builder.add_conditional_edges("check", route)
# 选中的节点会在下一个 superstep 中作为 PULL 任务执行
```

### 特点

- ✅ **响应式触发**：节点自动响应通道更新
- ✅ **数据流清晰**：通道 → 节点 → 通道
- ✅ **性能优化**：有触发优化和输入缓存
- ⚠️ **状态限制**：所有调用使用相同的状态结构

## 与通道的关系

### PregelNode 的通道属性

每个 `PregelNode` 有三个关键的通道相关属性：

```python
class PregelNode:
    channels: str | list[str]
    """节点读取的输入通道 - 用于准备输入数据"""
    
    triggers: list[str]
    """节点订阅的触发通道 - 当这些通道更新时，节点会被触发执行"""
    
    writers: list[Runnable]
    """节点写入的输出通道 - 通过 ChannelWrite 写入数据"""
```

### PULL 任务与通道

- **输入来源**：从节点的 `channels`（输入通道）读取数据
- **触发机制**：基于 `triggers`（触发通道）的版本变化
- **关键点**：`channels` 定义"读什么"，`triggers` 定义"何时触发"

### PUSH 任务与通道

- **输入来源**：**忽略**节点的 `channels` 属性，直接从 `Send.arg` 获取
- **触发机制**：基于 TASKS 通道中的 Send 对象
- **关键点**：可以传递任意数据，不受节点配置限制

### 输出通道

**两种任务都写入 output channels**，通过节点的 `writers` 统一处理：

```python
# 所有任务的写入都在 apply_writes() 函数中统一处理
for task in tasks:  # 包括 PUSH 和 PULL 任务
    for chan, val in task.writes:
        pending_writes_by_channel[chan].append(val)
```

## 使用场景对比

| 使用场景 | 使用 PULL | 使用 PUSH | 原因 |
|---------|-----------|-----------|------|
| **标准线性流程** | ✅ | ❌ | 固定边，数据流清晰 |
| **条件路由** | ✅ | ❌ | 基于状态选择下一个节点 |
| **Map-Reduce** | ❌ | ✅ | 动态数量，不同输入数据 |
| **并行处理列表** | ❌ | ✅ | 运行时决定调用次数 |
| **工具调用** | ✅ | ❌ | 基于消息状态触发 |
| **循环/迭代** | ✅ | ❌ | 基于状态变化触发 |

### 何时使用 PULL？

✅ **使用 PULL 当**：
1. **边是预定义的**：在编译时就知道可能的路径
2. **状态结构一致**：所有节点使用相同的状态结构
3. **响应式触发**：节点应该响应通道更新自动执行
4. **标准控制流**：线性、条件分支、循环等标准模式

**典型场景**：
- 标准工作流（A → B → C）
- 条件路由（if-else 分支）
- 工具调用链
- 状态机转换

### 何时使用 PUSH？

✅ **使用 PUSH 当**：
1. **动态数量**：运行时才知道要调用多少次
2. **不同输入**：每次调用需要不同的输入数据
3. **并行处理**：需要同时处理多个相同节点但不同输入
4. **灵活控制**：需要传递与当前状态无关的数据

**典型场景**：
- Map-Reduce 模式
- 并行处理列表
- 动态任务分发
- Fan-out/Fan-in 模式

## 执行流程

1. **当前 superstep (n)**：
   - 节点执行并可能写入 `TASKS` 通道（创建 Send 操作）→ 产生 PUSH 任务
   - 节点写入普通通道（更新通道版本）→ 产生 PULL 任务

2. **任务准备阶段**：
   - `prepare_next_tasks()` 被调用
   - 处理 PUSH 任务：从 `TASKS` 通道读取所有 Send 操作
   - 处理 PULL 任务：检查哪些节点应该被触发

3. **下一个 superstep (n+1)**：
   - PUSH 任务执行：将 Send 的消息传递给目标节点
   - PULL 任务执行：执行被触发的节点

## 数据流示例

### PULL 任务的标准流程

```
Superstep N:
  Node A 执行
    ├─ 读取: channels = ["input"] (从 input 通道读取)
    ├─ 写入: writers = [ChannelWrite("output")] (写入 output 通道)
    └─ 触发: triggers = ["output"] (output 更新触发 Node B)

Superstep N+1:
  prepare_next_tasks() 创建 PULL 任务:
    ├─ 检查: Node B 的 triggers = ["output"] 是否更新 → 是
    ├─ 读取: Node B 的 channels = ["output"] → 读取 output 通道的值
    └─ 创建: PULL 任务 (PULL, "node_b")

  Node B 执行 (PULL 任务)
    ├─ 输入: 来自 output 通道的值
    └─ 输出: 写入到其他通道
```

### PUSH 任务的动态流程

```
Superstep N:
  Node A 执行
    ├─ 读取: channels = ["data"]
    ├─ 写入: writers = [Send("node_b", {"custom": "data"})]
    └─ Send 对象写入 TASKS 通道

Superstep N+1:
  prepare_next_tasks() 创建 PUSH 任务:
    ├─ 读取: 从 TASKS 通道获取 Send 对象
    ├─ 忽略: Node B 的 channels 属性
    └─ 创建: PUSH 任务 (PUSH, idx)

  Node B 执行 (PUSH 任务)
    ├─ 输入: 来自 Send.arg = {"custom": "data"} (不是从通道读取)
    └─ 输出: 写入到 Node B 配置的输出通道
```

## 关键区别总结

| 维度 | PULL 任务 | PUSH 任务 |
|------|-----------|-----------|
| **输入来源** | 从节点的 `channels`（输入通道）读取 | 从 `Send.arg` 获取（忽略 `channels`） |
| **触发机制** | 基于 `triggers`（触发通道）的版本变化 | 基于 TASKS 通道中的 Send 对象 |
| **输出处理** | 通过节点的 `writers` 写入输出通道 | 通过目标节点的 `writers` 写入输出通道 |
| **数据流** | 通道 → 节点 → 通道 | Send → 节点 → 通道 |
| **灵活性** | 受节点配置限制 | 可传递任意数据 |
| **适用场景** | 标准工作流、条件路由、工具调用 | Map-Reduce、并行处理、动态分发 |
| **边定义** | 编译时预定义 | 运行时动态创建 |
| **调用数量** | 固定或基于条件 | 动态，可能很多 |
| **并行性** | 基于通道更新触发 | 显式并行调用 |

## 设计意图

这种设计的优势：

1. **PULL（响应式）**：
   - 节点自动响应通道更新
   - 数据流清晰：通道 → 节点 → 通道
   - 适合标准的数据流处理

2. **PUSH（命令式）**：
   - 显式控制节点调用
   - 可以传递与通道状态无关的数据
   - 适合动态路由和并行处理

3. **统一输出**：
   - 两种任务都通过相同的机制写入输出通道
   - 保证一致性，简化实现

## 总结

PUSH 和 PULL 任务是 LangGraph Pregel 执行模型的两个核心机制：

- **PUSH** 提供了显式的、动态的节点调用能力，支持复杂的控制流模式（Map-Reduce、并行处理）
- **PULL** 提供了基于数据流的自动触发机制，支持响应式编程模式（标准工作流、条件路由）

**选择原则**：
- **PULL**：预定义边、统一状态、响应式触发 → 标准工作流
- **PUSH**：动态数量、不同输入、并行处理 → Map-Reduce 和并行模式

两者结合使用，使得 LangGraph 能够支持从简单的线性流程到复杂的并行和条件执行的各种工作流模式。
