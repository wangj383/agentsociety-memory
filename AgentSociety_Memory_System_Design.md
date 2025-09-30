# AgentSociety Memory 系统整体设计文档

## 目录
1. [系统概述](#系统概述)
2. [架构设计](#架构设计)
3. [核心组件](#核心组件)
4. [数据流设计](#数据流设计)
5. [API 接口](#api-接口)
6. [使用示例](#使用示例)
7. [性能优化](#性能优化)
8. [扩展性设计](#扩展性设计)

---

## 系统概述

AgentSociety Memory 系统是一个为多智能体社会模拟设计的记忆管理系统。该系统采用分层架构设计，支持智能体的状态记忆、事件记忆和认知记忆，并提供语义搜索、时间过滤等高级功能。

### 核心特性
- **分层架构**：存储层、配置层、应用层分离
- **类型安全**：基于 Pydantic 的类型验证
- **向量化支持**：语义搜索和相似性匹配
- **并发安全**：异步编程和锁机制
- **可扩展性**：支持自定义记忆属性和行为块

---

## 架构设计

### 三层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    AgentSociety Memory 架构                     │
├─────────────────────────────────────────────────────────────────┤
│  🏙️  cityagent/ (应用层 - 城市智能体实现)                      │
│  ├── societyagent.py ──────┐                                   │
│  ├── firmagent.py ─────────┼─── 具体智能体实现                  │
│  ├── bankagent.py ────────┤                                   │
│  ├── governmentagent.py ───┤                                   │
│  ├── nbsagent.py ──────────┘                                   │
│  ├── memory_config.py ──────── 记忆配置定义                   │
│  └── blocks/ ──────────────── 行为块实现                      │
├─────────────────────────────────────────────────────────────────┤
│  🤖 agent/ (框架层 - 智能体核心框架)                          │
│  ├── agent_base.py ──────── Agent 基类定义                     │
│  ├── agent.py ───────────── 智能体基类实现                     │
│  ├── memory_config_generator.py ── 记忆配置生成器              │
│  ├── toolbox.py ─────────── 智能体工具箱                       │
│  ├── block.py ───────────── 行为块基类                        │
│  ├── context.py ─────────── 上下文管理                        │
│  └── dispatcher.py ──────── 行为分发器                        │
├─────────────────────────────────────────────────────────────────┤
│  🧠 memory/ (存储层 - 记忆系统实现)                           │
│  ├── memory.py ──────────── 记忆系统核心                       │
│  ├── KVMemory ──────────── 键值对记忆                         │
│  └── StreamMemory ──────── 流式记忆                           │
└─────────────────────────────────────────────────────────────────┘
```

### 依赖关系

```
cityagent/ ──继承──→ agent/ ──使用──→ memory/
     │                    │              │
     │                    │              │
     └──配置──→ memory_config.py ──使用──→ MemoryConfig
```

---

## 核心组件

### 1. Memory 存储层 (memory/)

#### KVMemory - 键值对记忆
```python
class KVMemory:
    """键值对记忆存储，用于存储智能体状态属性"""
    
    def __init__(self, memory_config: MemoryConfig, embedding: SparseTextEmbedding):
        self._memory_config = memory_config  # 记忆配置
        self._data = {}                      # 实际数据存储
        self._vectorstore = VectorStore(embedding)  # 向量存储
        self._key_to_doc_id = {}            # 键到文档ID映射
        self._lock = asyncio.Lock()          # 并发锁
    
    async def get(self, key: Any, default_value: Optional[Any] = None) -> Any:
        """获取记忆值"""
        
    async def update(self, key: Any, value: Any, mode: str = "replace") -> None:
        """更新记忆值，支持 replace 和 merge 模式"""
        
    async def search(self, query: str, top_k: int = 3, filter: Optional[dict] = None) -> str:
        """语义搜索记忆"""
```

#### StreamMemory - 流式记忆
```python
class StreamMemory:
    """流式记忆存储，用于存储时间序列事件"""
    
    def __init__(self, environment, status_memory, embedding, max_len=1000):
        self._memories: deque = deque(maxlen=max_len)  # 时间序列记忆
        self._memory_id_counter: int = 0              # 记忆ID计数器
        self._status_memory = status_memory           # 状态记忆引用
        self._vectorstore = VectorStore(embedding)    # 向量存储
    
    async def add(self, topic: str, description: str) -> int:
        """添加记忆节点"""
        
    async def search(self, query: str, topic: Optional[str] = None, 
                    top_k: int = 3, day_range: Optional[tuple] = None) -> str:
        """搜索流式记忆"""
        
    async def add_cognition_to_memory(self, memory_ids: list[int], cognition: str) -> None:
        """为记忆添加认知关联"""
```

#### Memory - 统一记忆管理器
```python
class Memory:
    """统一记忆管理器，整合 KVMemory 和 StreamMemory"""
    
    def __init__(self, environment, embedding, memory_config):
        self._status = KVMemory(memory_config=memory_config, embedding=embedding)
        self._stream = StreamMemory(environment=environment, embedding=embedding, 
                                  status_memory=self._status)
    
    @property
    def status(self) -> KVMemory:
        return self._status
    
    @property
    def stream(self) -> StreamMemory:
        return self._stream
```

### 2. Agent 配置层 (agent/)

#### MemoryAttribute - 记忆属性定义
```python
class MemoryAttribute(BaseModel):
    """记忆属性定义"""
    name: str                    # 属性名称
    type: Any                   # 属性类型
    default_or_value: Any        # 默认值或当前值
    description: str             # 属性描述
    whether_embedding: bool = False      # 是否需要向量化
    embedding_template: Optional[str] = None  # 向量化模板
```

#### MemoryConfig - 统一配置结构
```python
class MemoryConfig(BaseModel):
    """统一记忆配置结构"""
    attributes: dict[str, MemoryAttribute]  # 所有记忆属性的字典
    
    @staticmethod
    def from_list(attributes: list[MemoryAttribute]) -> "MemoryConfig":
        return MemoryConfig(attributes={attr.name: attr for attr in attributes})
```

### 3. CityAgent 应用层 (cityagent/)

#### 社会智能体记忆配置
```python
def memory_config_societyagent(distributions, class_config=None) -> MemoryConfig:
    """生成社会智能体记忆配置"""
    attributes = {
        # 基本属性
        "type": MemoryAttribute(name="type", type=str, default_or_value="citizen"),
        "name": MemoryAttribute(name="name", type=str, default_or_value="unknown", 
                               whether_embedding=True),
        "age": MemoryAttribute(name="age", type=int, default_or_value=20),
        
        # 需求满足度
        "hunger_satisfaction": MemoryAttribute(name="hunger_satisfaction", 
                                             type=float, default_or_value=0.9),
        "energy_satisfaction": MemoryAttribute(name="energy_satisfaction", 
                                               type=float, default_or_value=0.9),
        "safety_satisfaction": MemoryAttribute(name="safety_satisfaction", 
                                              type=float, default_or_value=0.4),
        "social_satisfaction": MemoryAttribute(name="social_satisfaction", 
                                              type=float, default_or_value=0.6),
        
        # 认知状态
        "emotion": MemoryAttribute(name="emotion", type=dict, 
                                  default_or_value={"sadness": 5, "joy": 5, ...}),
        "thought": MemoryAttribute(name="thought", type=str, 
                                 default_or_value="Currently nothing...", 
                                 whether_embedding=True),
        
        # 社会关系
        "social_network": MemoryAttribute(name="social_network", type=list, 
                                         default_or_value=[]),
        "chat_histories": MemoryAttribute(name="chat_histories", type=dict, 
                                         default_or_value={}),
        
        # 工作相关
        "firm_id": MemoryAttribute(name="firm_id", type=int, default_or_value=0),
        "work_hour_month": MemoryAttribute(name="work_hour_month", type=float, 
                                          default_or_value=160),
    }
    return MemoryConfig(attributes=attributes)
```

---

## 数据流设计

### 初始化流程
```
1. 配置阶段：
   cityagent/memory_config.py 
   ↓ 定义记忆属性
   agent/memory_config_generator.py 
   ↓ 生成配置结构
   memory/memory.py 
   ↓ 创建记忆实例

2. 智能体创建：
   cityagent/societyagent.py 
   ↓ 继承自
   agent/agent.py 
   ↓ 使用
   memory/memory.py
```

### 运行时数据流
```
智能体行为 → 记忆更新 → 状态管理 → 行为决策
     ↓           ↓          ↓         ↓
cityagent/ → memory/ → agent/ → cityagent/
```

### 记忆类型映射

| 记忆类型 | 存储位置 | 内容 | 用途 |
|---------|---------|------|------|
| **状态记忆** | KVMemory | 情绪、需求、关系强度 | 当前状态管理 |
| **事件记忆** | StreamMemory | 活动、对话、经历 | 历史回顾 |
| **认知记忆** | StreamMemory | 思考、反思、学习 | 认知更新 |
| **社交记忆** | StreamMemory | 对话历史、互动记录 | 关系维护 |

---

## API 接口

### Memory 核心接口

#### 状态记忆操作
```python
# 获取状态
emotion = await memory.status.get("emotion")
hunger = await memory.status.get("hunger_satisfaction")

# 更新状态
await memory.status.update("emotion", {"joy": 8, "sadness": 2})
await memory.status.update("hunger_satisfaction", 0.7)

# 语义搜索
results = await memory.status.search("happy moments", top_k=5)
```

#### 流式记忆操作
```python
# 添加事件记忆
memory_id = await memory.stream.add(
    topic="activity", 
    description="I went to the grocery store and bought food"
)

# 添加认知记忆
cognition_id = await memory.stream.add(
    topic="cognition", 
    description="I feel satisfied after completing my shopping"
)

# 关联认知记忆
await memory.stream.add_cognition_to_memory([memory_id], cognition_id)

# 搜索记忆
recent_activities = await memory.stream.search(
    query="shopping", 
    topic="activity", 
    top_k=10
)

# 搜索今天的记忆
today_memories = await memory.stream.search_today(query="work")
```

### 智能体集成接口

#### SocietyAgent 记忆使用
```python
class SocietyAgent(CitizenAgentBase):
    async def forward(self):
        # 获取当前状态
        emotion = await self.memory.status.get("emotion")
        hunger = await self.memory.status.get("hunger_satisfaction")
        
        # 更新状态
        await self.memory.status.update("thought", "I'm planning my day")
        
        # 添加活动记忆
        activity_id = await self.memory.stream.add(
            topic="activity",
            description="I decided to go shopping for groceries"
        )
        
        # 搜索相关记忆
        past_shopping = await self.memory.stream.search(
            query="shopping experience",
            top_k=3
        )
```

---

## 使用示例

### 1. 创建智能体记忆系统

```python
# 1. 创建记忆配置
memory_config = memory_config_societyagent(distributions)

# 2. 创建记忆实例
memory = Memory(
    environment=environment,
    embedding=embedding,
    memory_config=memory_config
)

# 3. 初始化向量化
await memory.initialize_embeddings()

# 4. 创建智能体
agent = SocietyAgent(
    id=1,
    name="Alice",
    toolbox=toolbox,
    memory=memory
)
```

### 2. 智能体行为与记忆交互

```python
# 智能体执行行为
async def agent_behavior():
    # 获取当前需求
    hunger = await agent.memory.status.get("hunger_satisfaction")
    
    if hunger < 0.3:
        # 记录决策
        await agent.memory.stream.add(
            topic="decision",
            description="I decided to eat because I'm hungry"
        )
        
        # 执行吃饭行为
        result = await agent.eat()
        
        # 更新状态
        await agent.memory.status.update("hunger_satisfaction", 0.9)
        
        # 记录结果
        await agent.memory.stream.add(
            topic="activity",
            description=f"I ate and now feel satisfied: {result}"
        )
```

### 3. 记忆搜索与决策

```python
# 基于记忆做决策
async def make_decision():
    # 搜索相关经验
    past_experiences = await agent.memory.stream.search(
        query="shopping at this store",
        top_k=5
    )
    
    # 搜索情绪状态
    happy_memories = await agent.memory.status.search(
        query="happy shopping",
        top_k=3
    )
    
    # 基于记忆做决策
    if "positive" in happy_memories:
        decision = "I'll go shopping because I had good experiences before"
    else:
        decision = "I'm hesitant about shopping due to past experiences"
    
    # 记录决策过程
    await agent.memory.stream.add(
        topic="cognition",
        description=f"Decision: {decision}. Based on: {past_experiences}"
    )
```

---

## 性能优化

### 1. 向量化优化
- **批量处理**：批量创建和更新向量嵌入
- **增量更新**：只更新变化的记忆向量
- **缓存机制**：缓存常用记忆的向量表示

### 2. 并发优化
- **异步操作**：所有记忆操作都是异步的
- **锁机制**：使用 `@lock_decorator` 确保线程安全
- **非阻塞I/O**：避免阻塞其他操作

### 3. 存储优化
- **容量限制**：StreamMemory 使用 `deque(maxlen=1000)` 限制内存使用
- **压缩存储**：对大型记忆进行压缩存储
- **索引优化**：为常用搜索字段建立索引

---

## 扩展性设计

### 1. 新增记忆类型
```python
# 在 cityagent/memory_config.py 中添加新的记忆属性
def memory_config_custom_agent():
    attributes = {
        # 现有属性...
        "custom_attribute": MemoryAttribute(
            name="custom_attribute",
            type=str,
            default_or_value="default_value",
            description="Custom memory attribute",
            whether_embedding=True
        )
    }
    return MemoryConfig(attributes=attributes)
```

### 2. 新增智能体类型
```python
# 在 cityagent/ 中创建新的智能体
class CustomAgent(CitizenAgentBase):
    ParamsType = CustomAgentConfig
    StatusAttributes = [
        # 自定义记忆属性
        MemoryAttribute(name="custom_field", ...),
        # ...
    ]
```

### 3. 新增行为块
```python
# 在 cityagent/blocks/ 中创建新的行为块
class CustomBlock(Block):
    async def forward(self, context):
        # 自定义行为逻辑
        pass
```

### 4. 自定义搜索策略
```python
# 扩展搜索功能
class AdvancedStreamMemory(StreamMemory):
    async def search_by_emotion(self, emotion_type: str, top_k: int = 5):
        """按情绪类型搜索记忆"""
        pass
    
    async def search_by_location(self, location: str, top_k: int = 5):
        """按位置搜索记忆"""
        pass
```

---

## 总结

AgentSociety Memory 系统通过分层架构设计，实现了：

1. **模块化**：存储层、配置层、应用层分离
2. **类型安全**：基于 Pydantic 的类型验证
3. **高性能**：异步编程和向量化搜索
4. **可扩展**：支持自定义记忆类型和智能体
5. **易用性**：简洁的 API 接口和丰富的功能

该系统为复杂的社会模拟提供了强大的记忆管理能力，支持智能体进行更加智能和人性化的行为决策。

---

*文档版本：1.0*  
*最后更新：2024年*
