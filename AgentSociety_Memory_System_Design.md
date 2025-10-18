# AgentSociety Memory 系统整体设计文档

## 目录
1. [系统概述](#1-系统概述)
2. [架构设计](#2-架构设计)
3. [核心组件](#3-核心组件)
   - 3.1 [Memory 存储层](#31-memory-存储层-memory)
   - 3.2 [Agent 配置层](#32-agent-配置层-agent)
   - 3.3 [CityAgent 应用层](#33-cityagent-应用层-cityagent)
4. [数据流设计](#4-数据流设计)
5. [记忆系统工作流程详解](#5-记忆系统工作流程详解)
6. [API 接口](#6-api-接口)
7. [使用示例](#7-使用示例)
8. [扩展性设计](#8-扩展性设计)
9. [案例研究](#9-案例研究)

---
# 1. 系统概述

## 🏗️ 系统架构核心

### Overview
- **双层记忆结构**：`KVMemory`（状态记忆）+ `StreamMemory`（流式记忆），由 `Memory` 聚合  
- **语义向量空间**：通过 FastEmbed + Qdrant 支持统一向量检索  
- **检索与认知循环**：支持语义检索、跨主题整合、认知处理  
- **工程特性**：异步接口、类型安全、模块化扩展  

---

### 双记忆存储架构

- **KVMemory（状态记忆）**  
  存储结构化的个体状态信息（如情绪、态度、画像属性），可直接更新与检索。  

- **StreamMemory（流记忆）**  
  存储时间序列事件（对话、行为、反思），按时间有序，并内置向量索引，用于相似性检索与主题化查询。  

- **统一向量空间**  
  所有记忆条目（无论 KV 还是 Stream）都会生成向量表示，统一进入向量存储，支持跨类型搜索与融合。  


```
KVMemory (status)          StreamMemory (stream)
═════════════════          ═══════════════════════
• Key-value pairs          • Time-ordered events
• No timestamp             • Has day + t (seconds)
• Mutable                  • Append-only
• Unlimited size           • Max 1000 items
• Current state            • Event history

Example:                   Example:
"occupation": "Physicist"  [Day 0, 09:00] "Went to lab"
"position": {aoi: 123}     [Day 0, 10:00] "Analyzed data"
"long_term_goals": [...]   [Day 0, 23:00] "Reflected on day"
```


---

### 智能检索与整合

- **语义搜索**：基于内容语义，而非关键词匹配  
- **跨主题检索**：支持在不同类别的记忆间寻找关联  
- **认知处理**：认知模块（如 `CognitionBlock`）调用记忆系统，检索主题相关事件 → LLM 推理 → 更新记忆，形成“记忆—认知—再记忆”的闭环  
- **时间维度**：流记忆天然按时间序列组织，可结合检索做时序过滤  

---

## 🚀 技术创新亮点

### Overview
- **FastEmbed + Qdrant**：高效语义嵌入与相似性搜索  
- **异步设计**：`async` 接口支持大规模智能体并发运行  
- **模块化结构**：`KVMemory` / `StreamMemory` / `VectorStore` / 配置生成器独立，便于扩展与替换  
- **类型安全**：核心数据结构基于 Pydantic，保证约束与可维护性  

---

### 记忆互通机制

- **统一存储**：不同主题与类型的记忆条目都在 `StreamMemory` 管理  
- **向量相似性**：通过向量索引（Qdrant）实现跨类型高效检索  
- **认知循环**：记忆被检索 → 输入认知模块 → 输出再写入记忆，形成闭环学习  

---

### 高性能特性

- **异步并发**：异步 API，降低 I/O 等待，提升系统规模化性能  
- **类型安全**：Pydantic 定义字段与配置，避免结构性错误  
- **可扩展性**：配置生成器可轻松添加不同智能体类型的记忆结构  
- **容量控制**：流式存储结合外部索引，保证可伸缩性（暂未内置“自然遗忘”机制）  

---

## 🔍 智能搜索能力（Intelligent Search Capabilities）

- **相似性搜索**（Similarity Search）：基于向量相似度找到语义相关记忆  
- **语义搜索**（Semantic Search）：理解上下文含义，而非简单关键词  
- **时间过滤**（Temporal Filtering）：利用流记忆的时序特征做检索筛选  
- **主题分类**（Topic Classification）：按记忆类型（如事件、对话、画像字段）组织和调用  

---

```
应用层 (cityagent/) → 框架层 (agent/) → 存储层 (memory/)
     ↓                    ↓                ↓
  具体智能体实现         智能体框架       记忆存储管理
  (Specific Agents)   (Generic Framework)  (Memory Storage)
```

---


# 2. 架构设计

## 整体架构：三层分离设计

记忆系统采用**三层分离**的架构设计，每一层都有明确的职责：

```
┌─────────────────────────────────────────────────────────────────┐
│                    AgentSociety Memory 架构                     │
├─────────────────────────────────────────────────────────────────┤
│  🏙️  cityagent/ (应用层 - 具体智能体实现)                      │
│  ├── societyagent.py ──────┐                                   │
│  ├── firmagent.py ─────────┼─── 不同类型的智能体                │
│  ├── bankagent.py ────────┤   (市民、企业、银行、政府等)        │
│  ├── governmentagent.py ───┤                                   │
│  ├── nbsagent.py ──────────┘                                   │
│  ├── memory_config.py ──────── 定义每种智能体的记忆属性        │
│  └── blocks/ ──────────────── 智能体的行为模块                 │
├─────────────────────────────────────────────────────────────────┤
│  🤖 agent/ (框架层 - 智能体通用框架)                          │
│  ├── agent_base.py ──────── 智能体基础类                      │
│  ├── agent.py ───────────── 智能体核心实现                     │
│  ├── memory_config_generator.py ── 记忆配置生成工具            │
│  ├── toolbox.py ─────────── 智能体工具集                       │
│  ├── block.py ───────────── 行为模块基类                      │
│  ├── context.py ─────────── 上下文管理器                       │
│  └── dispatcher.py ──────── 行为调度器                        │
├─────────────────────────────────────────────────────────────────┤
│  🧠 memory/ (存储层 - 记忆系统核心)                           │
│  ├── memory.py ──────────── 记忆系统主控制器                   │
│  ├── KVMemory ──────────── 状态记忆存储                       │
│  └── StreamMemory ──────── 事件记忆存储                       │
└─────────────────────────────────────────────────────────────────┘
```

### 各层职责说明

#### 🏙️ 应用层 (cityagent/)
**作用**：实现具体的智能体类型
- **societyagent.py**：社会智能体（市民）
- **firmagent.py**：企业智能体
- **bankagent.py**：银行智能体
- **governmentagent.py**：政府智能体
- **memory_config.py**：定义每种智能体需要记住什么

#### 🤖 框架层 (agent/)
**作用**：提供智能体的通用功能
- **agent_base.py**：智能体的基础定义
- **agent.py**：智能体的核心实现
- **memory_config_generator.py**：帮助生成记忆配置
- **block.py**：行为模块的基础类

#### 🧠 存储层 (memory/)
**作用**：管理智能体的记忆
- **memory.py**：记忆系统的主控制器
- **KVMemory**：存储智能体的状态（如情绪、需求）
- **StreamMemory**：存储智能体的经历（如活动、对话）

### 数据流向
Example
```
智能体状态 → 记忆读取 → 行为决策 → 行为执行 → 记忆写入 → 状态更新
    ↑                                                      ↓
    └─────────────── 循环影响 ←─────────────────────────────┘
```

**实际流程**（Based on Code）：
1. **智能体执行**：`SocietyAgent.forward()` → 执行Blocks（NeedsBlock, PlanBlock, SocialBlock等）
2. **记忆记录**：`memory.stream.add(topic="social", description="...")` → 记录行为到StreamMemory
3. **状态更新**：`memory.status.update("current_plan", plan)` → 更新状态到KVMemory  
4. **记忆检索**：`memory.stream.search(query="...")` → 基于历史记忆做决策
5. **循环影响**：记忆影响下次行为选择，形成持续学习循环

---

## 3. 核心组件

### 3.1 Memory 存储层 (memory/) 

#### 🎯 系统总览

```
AgentSociety Memory 系统架构
├── Memory (主控制器)
│   ├── KVMemory (状态记忆) - 存储智能体当前状态
│   │   ├── 情绪状态 (emotion)
│   │   ├── 需求满足度 (satisfaction) 
│   │   ├── 个人属性 (personal_info)
│   │   └── 关系网络 (social_network)
│   └── StreamMemory (流记忆) - 存储智能体经历事件
│       ├── topic="social" (社交行为记忆)
│       ├── topic="mobility" (移动行为记忆)
│       ├── topic="economy" (经济行为记忆)
│       ├── topic="activity" (活动行为记忆)
│       ├── topic="cognition" (认知处理记忆)
│       └── topic="other" (其他行为记忆)
└── VectorStore (向量存储) - 支持语义搜索
    ├── 相似性搜索 (similarity_search)
    ├── 时间过滤 (day_range filter)
    └── 主题过滤 (topic filter)
```

#### ⚠️ 记忆互通与语义整合机制

**不同Action Space的记忆是互通的**，通过以下机制实现语义整合：

```
记忆存储与检索机制
├── 统一存储 (Unified Storage)
│   └── 所有topic的记忆存储在同一个StreamMemory中
├── 标签分类 (Topic Classification)
│   ├── topic="social" → 社交行为记忆
│   ├── topic="mobility" → 移动行为记忆
│   ├── topic="economy" → 经济行为记忆
│   ├── topic="activity" → 活动行为记忆
│   ├── topic="cognition" → 认知处理记忆
│   └── topic="other" → 其他行为记忆
├── 语义检索 (Semantic Retrieval)
│   ├── 跨topic搜索：基于内容语义而非标签
│   ├── 向量相似性：所有记忆映射到统一向量空间
│   └── 关联整合：认知记忆可关联多个topic的记忆
├── 智能检索策略 (Intelligent Retrieval)
│   ├── 语义搜索：理解记忆内容含义
│   ├── 标签过滤：可选择性按topic过滤
│   └── 时间过滤：支持时间范围检索
└── 决策整合 (Decision Integration)
    ├── 综合所有类型记忆做决策
    ├── 跨行为类型的经验学习
    └── 语义理解驱动的行为选择
```
#### 记忆互通流程
智能体行为 → 记忆记录 → 向量化存储 → 语义搜索 → 决策影响
     ↑                                              ↓
     └────────── 循环学习 ←─────────────────────────┘

#### 📊 记忆类型映射

| 记忆类型 | 存储位置 | 内容 | 用途 |
|---------|---------|------|------|
| **状态记忆** | KVMemory | 情绪、需求、关系强度 | 当前状态管理 |
| **事件记忆** | StreamMemory | 活动、对话、经历 | 历史回顾 |
| **认知记忆** | StreamMemory | 思考、反思、学习 | 认知更新 |
| **社交记忆** | StreamMemory | 对话历史、互动记录 | 关系维护 |

---


#### 🗂️ KVMemory - 状态记忆存储（Status Memory Storage）
**作用**：存储智能体的当前状态，就像人类的"当前状态"（Current State）

**存储内容**（Stored Content）：
- 情绪状态（Emotional State）：`{"joy": 8, "sadness": 2, "anger": 1}`
- 需求满足度（Need Satisfaction）：`{"hunger_satisfaction": 0.7, "social_satisfaction": 0.9}`
- 个人信息（Personal Information）：`{"name": "Alice", "age": 25, "occupation": "工程师"}`
- 关系强度（Relationship Strength）：`{"friends": ["Bob", "Charlie"], "relationships": {"Bob": 0.8}}`

**特点**（Features）：
- 键值对存储（Key-Value Storage），快速访问
- 支持相似性搜索（Similarity Search）（如搜索"快乐时刻"）
- 支持嵌入生成（Embedding Generation）和向量化存储
- 支持更新和合并操作（Update & Merge Operations）

```python
class KVMemory:
    """状态记忆存储 - 存储智能体的当前状态和属性"""
    
    def __init__(self, memory_config: MemoryConfig, embedding: SparseTextEmbedding):
        self._memory_config = memory_config  # 记忆配置
        self._data = {}                      # 实际数据存储
        self._vectorstore = VectorStore(embedding)  # 向量存储
        self._key_to_doc_id = {}            # 键到文档ID映射
        self._lock = asyncio.Lock()          # 并发锁
    
    async def get(self, key: Any, default_value: Optional[Any] = None) -> Any:
        """
        获取记忆值
        
        Args:
            key: 要检索的键名
            default_value: 如果键不存在时返回的默认值
            
        Returns:
            检索到的值，如果键不存在且未提供默认值则抛出KeyError
            
        Raises:
            KeyError: 当键不存在且未提供默认值时
        """
        if key in self._data:
            return deepcopy(self._data[key])
        else:
            if default_value is None:
                raise KeyError(f"No attribute `{key}` in memories!")
            else:
                return default_value
        
    async def update(self, key: Any, value: Any, mode: str = "replace") -> None:
        """
        更新记忆值，支持 replace 和 merge 模式
        
        Args:
            key: 要更新的键名
            value: 新的值
            mode: 更新模式，"replace"为替换，"merge"为合并
            
        Raises:
            ValueError: 当提供无效的更新模式时
            KeyError: 当键不存在时（仅在merge模式下）
        """
        # 如果键不存在，直接添加
        if key not in self._data:
            self._data[key] = value
            # 检查是否需要嵌入
            if self.should_embed(key):
                semantic_text = self._generate_semantic_text(key, value)
                doc_ids = await self._vectorstore.add_documents(
                    documents=[semantic_text],
                    extra_tags={"key": key}
                )
                self._key_to_doc_id[key] = doc_ids[0]
            return
        
        # 更新现有键
        if mode == "replace":
            self._data[key] = value
            # 更新嵌入
            if self.should_embed(key):
                await self._update_embeddings(key, value)
        elif mode == "merge":
            # 根据类型进行合并
            original_value = self._data[key]
            if isinstance(original_value, set):
                original_value.update(set(value))
            elif isinstance(original_value, dict):
                original_value.update(dict(value))
            elif isinstance(original_value, list):
                original_value.extend(list(value))
            # 更新嵌入
            if self.should_embed(key):
                await self._update_embeddings(key, self._data[key])
        else:
            raise ValueError(f"Invalid update mode `{mode}`!")
        
    async def search(self, query: str, top_k: int = 3, filter: Optional[dict] = None) -> str:
        """
        语义搜索记忆
        
        Args:
            query: 搜索查询文本
            top_k: 返回最相关结果的数量
            filter: 额外的过滤条件
            
        Returns:
            格式化的搜索结果字符串，如果没有结果返回"Nothing"
        """
        filter_dict = {}
        if filter is not None:
            filter_dict.update(filter)
        
        # 执行向量相似性搜索
        top_results: list[tuple[str, float, dict]] = (
            await self._vectorstore.similarity_search(
                query=query,
                k=top_k,
                filter=filter_dict,
            )
        )
        
        # 格式化结果
        formatted_results = []
        for content, score, metadata in top_results:
            formatted_results.append(f"- {content}")
        
        return "Nothing" if len(formatted_results) == 0 else "\n".join(formatted_results)
```

#### 📅 StreamMemory - 流记忆存储（Stream Memory Storage）
**作用**：存储智能体的经历和事件，就像人类的"回忆"（Memories）

**存储内容**（Stored Content）：
- 活动记录（Activity Records）：`"我去商店购物了"`
- 对话内容（Conversation Content）：`"和Bob讨论了工作问题"`
- 认知反思（Cognitive Reflection）：`"我觉得今天过得很充实"`
- 社交互动（Social Interaction）：`"在聚会上认识了新朋友"`

**特点**（Features）：
- 按时间顺序存储（Time-Ordered Storage）
- 支持主题分类（Topic Classification）（活动、社交、工作等）
- 支持相似性搜索（Similarity Search）和时间范围过滤
- 支持认知关联（Cognitive Association）

```python
class StreamMemory:
    """事件记忆存储 - 存储智能体的经历和事件"""
    
    def __init__(self, environment, status_memory, embedding, max_len=1000):
        self._memories: deque = deque(maxlen=max_len)  # 时间序列记忆
        self._memory_id_counter: int = 0              # 记忆ID计数器
        self._status_memory = status_memory           # 状态记忆引用
        self._vectorstore = VectorStore(embedding)    # 向量存储
    
    async def add(self, topic: str, description: str) -> int:
        """
        添加记忆节点到流式记忆
        
        Args:
            topic: 记忆主题（如"activity", "cognition", "social"等）
            description: 记忆描述文本
            
        Returns:
            新创建的记忆节点ID
            
        Raises:
            ValueError: 当环境未初始化时
        """
        if self._environment is None:
            raise ValueError("Environment is not initialized")
        
        # 获取时间和位置信息
        day, t = self._environment.get_datetime()
        position = await self._status_memory.get("position")
        if "aoi_position" in position:
            location = position["aoi_position"]["aoi_id"]
        elif "lane_position" in position:
            location = position["lane_position"]["lane_id"]
        else:
            location = "unknown"
        
        # 创建记忆节点
        current_id = self._memory_id_counter
        self._memory_id_counter += 1
        memory_node = MemoryNode(
            topic=topic,
            day=day,
            t=t,
            location=location,
            description=description,
            id=current_id,
        )
        self._memories.append(memory_node)
        
        # 创建向量嵌入
        await self._vectorstore.add_documents(
            documents=[description],
            extra_tags={
                "topic": topic,
                "day": day,
                "time": t,
            },
        )
        
        return current_id
        
    async def search(self, query: str, topic: Optional[str] = None, 
                    top_k: int = 3, day_range: Optional[tuple] = None) -> str:
        """
        搜索流式记忆
        
        Args:
            query: 搜索查询文本
            topic: 可选的记忆主题过滤
            top_k: 返回最相关结果的数量
            day_range: 可选的时间范围过滤 (start_day, end_day)
            
        Returns:
            格式化的搜索结果字符串，按时间倒序排列
        """
        filter_dict: dict[str, Any] = {"type": "stream"}
        
        if topic:
            filter_dict["topic"] = topic
        
        # 添加时间范围过滤
        if day_range:
            start_day, end_day = day_range
            filter_dict["day"] = {"gte": start_day, "lte": end_day}
        
        # 执行向量搜索
        top_results = await self._vectorstore.similarity_search(
            query=query,
            k=top_k,
            filter=filter_dict,
        )
        
        # 按时间排序结果
        sorted_results = sorted(
            top_results,
            key=lambda x: (x[2].get("day", 0), x[2].get("time", 0)),
            reverse=True,
        )
        
        # 格式化结果
        formatted_results = []
        for content, score, metadata in sorted_results:
            memory_topic = metadata.get("topic", "unknown")
            memory_day = metadata.get("day", "unknown")
            memory_time_seconds = metadata.get("time", "unknown")
            
            # 格式化时间
            if memory_time_seconds != "unknown":
                hours = memory_time_seconds // 3600
                minutes = (memory_time_seconds % 3600) // 60
                seconds = memory_time_seconds % 60
                memory_time = f"{hours:02d}:{minutes:02d}:{seconds:02d}"
            else:
                memory_time = "unknown"
            
            memory_location = metadata.get("location", "unknown")
            
            formatted_results.append(
                f"- [{memory_topic}]: {content} [day: {memory_day}, time: {memory_time}, "
                f"location: {memory_location}]"
            )
        
        return "Nothing" if len(formatted_results) == 0 else "\n".join(formatted_results)
        
    async def add_cognition_to_memory(self, memory_ids: list[int], cognition: str) -> None:
        """
        为记忆添加认知关联
        
        Args:
            memory_ids: 要关联的记忆ID列表
            cognition: 认知描述文本
        """
        # 查找对应的记忆
        target_memories = []
        for memory in self._memories:
            if memory.id in memory_ids:
                target_memories.append(memory)
        
        # 添加认知记忆
        cognition_id = await self.add(topic="cognition", description=cognition)
        
        # 更新原始记忆的认知ID
        for target_memory in target_memories:
            target_memory.cognition_id = cognition_id
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

### 3.2 Agent 配置层 (agent/)

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

### 3.3 CityAgent 应用层 (cityagent/)

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

## 4. 数据流设计

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
---

## 🔄 最小工作流程（Minimal Example）

### Environment 感知
智能体从环境接收一个新事件（如对话内容、观察结果）。  
这一步就是产生一条“经验”（memory item）。  

### Insert → Storage
- 事件通过 `StreamMemory.add_memory_node()` 插入流记忆。  
- 如果是属性更新（情绪/态度），则通过 `KVMemory.update()` 写入状态记忆。  

### Merging / Replace（可选）
若新信息与已有记忆高度相似，可以选择更新原条目，而不是简单追加。  
（在仓库里，这通常由上层逻辑控制，例如认知块根据检索结果决定更新/覆盖。）  

### Query / Retrieval
当智能体需要决策时，会构造查询上下文，调用  
`VectorStore.similarity_search()` 或 `StreamMemory.get_related_cognition()` 检索相关记忆。  

### To Action
检索到的记忆被送入 **认知模块（如 CognitionBlock）**，  
经由 LLM 推理 → 生成计划/更新态度 → 输出行为或写回记忆。  
行为结果再反馈到环境，循环开始下一轮。  

---

## 📄 最小代码骨架（基于仓库 API）

```python
from agentsociety.memory.memory import Memory
from agentsociety.agent.memory_config_generator import default_memory_config_citizen
from fastembed import SparseTextEmbedding

# 1) 初始化
embedding = SparseTextEmbedding()
# day, t = self._environment.get_datetime()

class MyEnvironment:
    def __init__(self):
        pass

    def get_datetime(self):
        # 0,0: day, time
        return 0, 0

my_environment_obj = MyEnvironment()

memory = Memory(environment=my_environment_obj, embedding=embedding, memory_config=default_memory_config_citizen())
await memory.initialize_embeddings()

# 2) Insert: 从环境接收到新事件，写入流记忆
event_id = await memory.stream.add_memory_node(
    text="Talked with neighbor about traffic jam",
    topic="social"
)

# 3) Update: 更新状态记忆
await memory.status.update({"mood": "frustrated"})

# 4) Retrieval: 按上下文查询相关记忆
hits = await memory.stream._vectorstore.similarity_search(query="commute issue", k=3)

# 5) Action: 将检索结果传给 CognitionBlock / LLM，生成行动或态度更新
# （伪代码）
# cognition_block = CognitionBlock(...)
# action = await cognition_block.run(memory, hits)
```

---

## 5. 记忆系统工作流程详解

### 1. 系统架构概览

AgentSociety记忆系统采用分层架构设计：

```
Memory (主控制器)
├── KVMemory (状态记忆)
│   ├── 键值对存储
│   ├── 向量存储集成
│   └── 异步锁机制
└── StreamMemory (流记忆)
    ├── 时间序列存储
    ├── 向量搜索
    └── 认知关联
```

### 2. 记忆插入（Insert）流程

#### 2.1 状态记忆插入
```python
# 通过KVMemory.update()方法
async def update(self, key: Any, value: Any, mode: Union["replace", "merge"]):
    # 1. 检查键是否存在
    if key not in self._data:
        self._data[key] = value
        # 2. 检查是否需要嵌入
        if self.should_embed(key):
            semantic_text = self._generate_semantic_text(key, value)
            # 3. 添加到向量存储
            doc_ids = await self._vectorstore.add_documents(...)
            self._key_to_doc_id[key] = doc_ids[0]
```

#### 2.2 流记忆插入
```python
# 通过StreamMemory.add()方法
async def add(self, topic: str, description: str) -> int:
    # 1. 获取时间戳和位置信息
    day, t = self._environment.get_datetime()
    position = await self._status_memory.get("position")
    
    # 2. 创建记忆节点
    memory_node = MemoryNode(
        topic=topic, day=day, t=t, 
        location=location, description=description, id=current_id
    )
    
    # 3. 添加到内存队列
    self._memories.append(memory_node)
    
    # 4. 创建向量嵌入
    await self._vectorstore.add_documents(
        documents=[description],
        extra_tags={"topic": topic, "day": day, "time": t}
    )
```

### 3. 编码（Encoding）流程

#### 3.1 语义文本生成
```python
def _generate_semantic_text(self, key: str, value: Any) -> str:
    if key in self._memory_config.attributes:
        config = self._memory_config.attributes[key]
        if config.embedding_template:
            return config.embedding_template.format(value)
    return f"My {key} is {value}"
```

#### 3.2 向量嵌入生成
```python
# 使用FastEmbed进行稀疏向量嵌入
embeddings = self._embeddings.embed(documents)
# 创建Qdrant点结构
points.append(models.PointStruct(
    id=point_id,
    vector={"text-sparse": models.SparseVector(
        indices=embedding.indices.tolist(),
        values=embedding.values.tolist()
    )},
    payload=metadata
))
```

### 4. 存储（Storage）流程

#### 4.1 内存存储
- **KVMemory**: 使用Python字典存储键值对
- **StreamMemory**: 使用deque(maxlen=1000)限制最大存储

#### 4.2 向量存储
```python
# 使用Qdrant向量数据库
self._client.create_collection(
    collection_name=self._collection_name,
    vectors_config=models.VectorParams(size=384, distance=Distance.COSINE),
    sparse_vectors_config={
        "text-sparse": models.SparseVectorParams(
            index=models.SparseIndexParams(on_disk=False)
        )
    }
)
```

#### 4.3 VectorStore 核心方法实现

```python
class VectorStore:
    """向量存储实现，使用Qdrant和FastEmbed"""
    
    async def add_documents(self, documents: list[str], extra_tags: Optional[dict] = None) -> list[str]:
        """
        添加文档到向量存储
        
        Args:
            documents: 要添加的文档列表
            extra_tags: 额外的元数据标签
            
        Returns:
            添加的文档ID列表
        """
        # 生成嵌入
        embeddings = self._embeddings.embed(documents)
        
        # 准备Qdrant点
        points = []
        for i, (doc, embedding) in enumerate(zip(documents, embeddings)):
            metadata = {"content": doc}
            if extra_tags is not None:
                metadata.update(extra_tags)
            
            point_id = str(uuid.uuid4())
            points.append(models.PointStruct(
                id=point_id,
                vector={"text-sparse": models.SparseVector(
                    indices=embedding.indices.tolist(),
                    values=embedding.values.tolist()
                )},
                payload=metadata
            ))
        
        # 上传到Qdrant
        self._client.upsert(collection_name=self._collection_name, points=points)
        return [point.id for point in points]
    
    async def similarity_search(self, query: str, k: int = 4, filter: Optional[dict] = None) -> list[tuple[str, float, dict]]:
        """
        执行相似性搜索
        
        Args:
            query: 查询文本
            k: 返回结果数量
            filter: 过滤条件
            
        Returns:
            (内容, 相似度分数, 元数据) 元组列表
        """
        # 生成查询嵌入
        query_embedding = list(self._embeddings.query_embed(query))[0]
        
        # 构建过滤条件
        search_filter = {}
        if filter is not None:
            search_filter.update(filter)
        
        # 创建命名稀疏向量
        named_vector = models.NamedSparseVector(
            name="text-sparse",
            vector=models.SparseVector(
                indices=query_embedding.indices.tolist(),
                values=query_embedding.values.tolist()
            )
        )
        
        # 构建过滤条件
        must_conditions = []
        for key, value in search_filter.items():
            if isinstance(value, dict):
                must_conditions.append(
                    models.FieldCondition(
                        key=key,
                        range=models.Range(
                            gte=value.get("gte"),
                            lte=value.get("lte")
                        )
                    )
                )
            else:
                must_conditions.append(
                    models.FieldCondition(key=key, match=models.MatchValue(value=value))
                )
        
        # 执行搜索
        search_result = self._client.search(
            collection_name=self._collection_name,
            query_vector=named_vector,
            limit=k,
            query_filter=models.Filter(must=must_conditions)
        )
        
        # 格式化结果
        results = []
        for hit in search_result:
            content = hit.payload.get("content", "")
            score = hit.score
            metadata = {k: v for k, v in hit.payload.items() if k != "content"}
            results.append((content, score, metadata))
        
        return results
```

### 5. 检索（Retrieve）流程

#### 5.1 状态记忆检索
```python
async def get(self, key: Any, default_value: Optional[Any] = None) -> Any:
    if key in self._data:
        return deepcopy(self._data[key])
    else:
        if default_value is None:
            raise KeyError(f"No attribute `{key}` in memories!")
        else:
            return default_value
```

#### 5.2 流记忆检索
```python
async def search(self, query: str, topic: Optional[str] = None, 
                top_k: int = 3, day_range: Optional[tuple] = None) -> str:
    # 1. 构建过滤条件
    filter_dict = {"type": "stream"}
    if topic: filter_dict["topic"] = topic
    if day_range: filter_dict["day"] = {"gte": start_day, "lte": end_day}
    
    # 2. 执行向量相似性搜索
    top_results = await self._vectorstore.similarity_search(
        query=query, k=top_k, filter=filter_dict
    )
    
    # 3. 格式化结果
    return self.format_memory(sorted_results)
```

### 6. 更新（Update）流程

#### 6.1 替换模式更新
```python
if mode == "replace":
    self._data[key] = value
    if self.should_embed(key):
        # 删除旧嵌入
        await self._vectorstore.delete_documents([old_doc_id])
        # 添加新嵌入
        doc_ids = await self._vectorstore.add_documents([new_semantic_text])
```

#### 6.2 合并模式更新
```python
elif mode == "merge":
    if isinstance(original_value, set):
        original_value.update(set(value))
    elif isinstance(original_value, dict):
        original_value.update(dict(value))
    elif isinstance(original_value, list):
        original_value.extend(list(value))
    # 更新嵌入
    await self._update_embeddings(key, self._data[key])
```

### 7. 遗忘（Forgetting）机制

#### 7.1 容量限制遗忘
```python
# StreamMemory使用deque限制
self._memories: deque = deque(maxlen=max_len)  # 默认1000条
```

### 8. 认知处理流程（外部+memory）

#### 8.1 态度更新
```python
async def attitude_update(self):
    """
    更新智能体对特定话题的态度
    
    工作流程:
    1. 获取当前态度状态
    2. 搜索相关历史事件
    3. 构建LLM提示词
    4. 生成新的态度评分
    5. 更新记忆中的态度
    
    Args:
        无直接参数，从memory中获取当前状态
        
    Returns:
        None，直接更新memory中的attitude字段
    """
    # 1. 获取当前态度
    attitude = await self.memory.status.get("attitude")
    
    # 2. 获取智能体基本信息
    prompt_data = {
        "gender": await self.memory.status.get("gender"),
        "age": await self.memory.status.get("age"),
        "race": await self.memory.status.get("race"),
        "religion": await self.memory.status.get("religion"),
        "marriage_status": await self.memory.status.get("marriage_status"),
        "residence": await self.memory.status.get("residence"),
        "occupation": await self.memory.status.get("occupation"),
        "education": await self.memory.status.get("education"),
        "personality": await self.memory.status.get("personality"),
        "consumption": await self.memory.status.get("consumption"),
        "family_consumption": await self.memory.status.get("family_consumption"),
        "income": await self.memory.status.get("income"),
        "skill": await self.memory.status.get("skill"),
        "thought": await self.memory.status.get("thought"),
        "emotion_types": await self.memory.status.get("emotion_types"),
    }
    
    # 3. 对每个态度话题进行更新
    for topic in attitude:
        # 搜索相关事件
        incident_str = await self.memory.stream.search(query=topic, top_k=20)
        
        # 构建提示词
        description_prompt = f"""
        You are a {prompt_data['gender']}, aged {prompt_data['age']}, belonging to the {prompt_data['race']} race and identifying as {prompt_data['religion']}. 
        Your marital status is {prompt_data['marriage_status']}, and you currently reside in a {prompt_data['residence']} area. 
        Your occupation is {prompt_data['occupation']}, and your education level is {prompt_data['education']}. 
        You are {prompt_data['personality']}, with a consumption level of {prompt_data['consumption']} and a family consumption level of {prompt_data['family_consumption']}. 
        Your income is {prompt_data['income']}, and you are skilled in {prompt_data['skill']}.
        My current emotion intensities are (0 meaning not at all, 10 meaning very much):
        sadness: {prompt_data['sadness']}, joy: {prompt_data['joy']}, fear: {prompt_data['fear']}, disgust: {prompt_data['disgust']}, anger: {prompt_data['anger']}, surprise: {prompt_data['surprise']}.
        You have the following thoughts: {prompt_data['thought']}.
        In the following 21 words, I have chosen {prompt_data['emotion_types']} to represent your current status:
        Joy, Distress, Resentment, Pity, Hope, Fear, Satisfaction, Relief, Disappointment, Pride, Admiration, Shame, Reproach, Liking, Disliking, Gratitude, Anger, Gratification, Remorse, Love, Hate.
        """
        
        # 添加事件信息
        if incident_str:
            incident_prompt = "Today, these incidents happened:" + incident_str
        else:
            incident_prompt = "No incidents happened today."
        
        # 构建问题
        previous_attitude = str(attitude[topic])
        problem_prompt = (
            f"You need to decide your attitude towards topic: {topic}, "
            f"which you previously rated your attitude towards this topic as: {previous_attitude} "
            "(0 meaning oppose, 10 meaning support). "
            'Please return a new attitude rating (0-10, smaller meaning oppose, larger meaning support) in JSON format, and explain, e.g. {{"attitude": 5}}'
        )
        
        # 4. 调用LLM生成新态度
        question_prompt = description_prompt + incident_prompt + problem_prompt
        question_prompt = FormatPrompt(question_prompt)
        
        # 添加情感数据
        emotion = await self.memory.status.get("emotion")
        prompt_data.update({
            "sadness": emotion["sadness"],
            "joy": emotion["joy"],
            "fear": emotion["fear"],
            "disgust": emotion["disgust"],
            "anger": emotion["anger"],
            "surprise": emotion["surprise"]
        })
        
        await question_prompt.format(**prompt_data)
        
        # 重试机制
        evaluation = True
        response = {}
        for retry in range(10):
            try:
                _response = await self.llm.atext_request(
                    question_prompt.to_dialog(),
                    timeout=300,
                    response_format={"type": "json_object"}
                )
                json_str = extract_json(_response)
                if json_str:
                    response = json_repair.loads(json_str)
                    evaluation = False
                    break
            except Exception:
                pass
        
        if evaluation:
            raise Exception(f"Request for attitude:{topic} update failed")
        
        # 5. 更新态度
        attitude[topic] = response["attitude"]
    
    # 保存更新后的态度
    await self.memory.status.update("attitude", attitude)
```

#### 8.2 思维更新
```python
async def thought_update(self):
    """
    生成智能体的日常反思和思维更新
    
    工作流程:
    1. 搜索今日发生的事件
    2. 构建反思提示词
    3. 调用LLM生成思维总结
    4. 更新思维状态和认知记忆
    
    Returns:
        str: 生成的思维描述
    """
    # 1. 构建基本信息提示
    description_prompt = """
    You are a {gender}, aged {age}, belonging to the {race} race and identifying as {religion}. 
    Your marital status is {marriage_status}, and you currently reside in a {residence} area. 
    Your occupation is {occupation}, and your education level is {education}. 
    You are {personality}, with a consumption level of {consumption} and a family consumption level of {family_consumption}. 
    Your income is {income}, and you are skilled in {skill}.
    My current emotion intensities are (0 meaning not at all, 10 meaning very much):
    sadness: {sadness}, joy: {joy}, fear: {fear}, disgust: {disgust}, anger: {anger}, surprise: {surprise}.
    You have the following thoughts: {thought}.
    In the following 21 words, I have chosen {emotion_types} to represent your current status:
    Joy, Distress, Resentment, Pity, Hope, Fear, Satisfaction, Relief, Disappointment, Pride, Admiration, Shame, Reproach, Liking, Disliking, Gratitude, Anger, Gratification, Remorse, Love, Hate.
    """
    
    # 2. 搜索今日事件
    incident_str = await self.memory.stream.search_today(top_k=20)
    if incident_str:
        incident_prompt = "Today, these incidents happened:\n" + incident_str
    else:
        incident_prompt = "No incidents happened today."
    
    # 3. 构建反思问题
    question_prompt = """
        Please review what happened today and share your thoughts and feelings about it.
        Consider your current emotional state and experiences, then:
        1. Summarize your thoughts and reflections on today's events
        2. Choose one word that best describes your current emotional state from: Joy, Distress, Resentment, Pity, Hope, Fear, Satisfaction, Relief, Disappointment, Pride, Admiration, Shame, Reproach, Liking, Disliking, Gratitude, Anger, Gratification, Remorse, Love, Hate.
        Return in JSON format, e.g. {{"thought": "Currently nothing good or bad is happening, I think ...."}}
    """
    
    # 4. 组合完整提示词
    question_prompt = description_prompt + incident_prompt + question_prompt
    question_prompt = FormatPrompt(question_prompt)
    
    # 5. 获取情感数据
    emotion = await self.memory.status.get("emotion")
    await question_prompt.format(
        gender=await self.memory.status.get("gender"),
        age=await self.memory.status.get("age"),
        race=await self.memory.status.get("race"),
        religion=await self.memory.status.get("religion"),
        marriage_status=await self.memory.status.get("marriage_status"),
        residence=await self.memory.status.get("residence"),
        occupation=await self.memory.status.get("occupation"),
        education=await self.memory.status.get("education"),
        personality=await self.memory.status.get("personality"),
        consumption=await self.memory.status.get("consumption"),
        family_consumption=await self.memory.status.get("family_consumption"),
        income=await self.memory.status.get("income"),
        skill=await self.memory.status.get("skill"),
        sadness=emotion["sadness"],
        joy=emotion["joy"],
        fear=emotion["fear"],
        disgust=emotion["disgust"],
        anger=emotion["anger"],
        surprise=emotion["surprise"],
        thought=await self.memory.status.get("thought"),
        emotion_types=await self.memory.status.get("emotion_types")
    )
    
    # 6. 调用LLM生成思维
    evaluation = True
    response = {}
    for retry in range(10):
        try:
            _response = await self.llm.atext_request(
                question_prompt.to_dialog(),
                timeout=300,
                response_format={"type": "json_object"}
            )
            json_str = extract_json(_response)
            if json_str:
                response = json_repair.loads(json_str)
                evaluation = False
                break
        except Exception:
            pass
    
    if evaluation:
        raise Exception("Request for cognition update failed")
    
    # 7. 更新思维状态
    thought = str(response["thought"])
    await self.memory.status.update("thought", thought)
    
    # 8. 添加认知记忆
    await self.memory.stream.add(topic="cognition", description=thought)
    
    return thought
```

### 9. 记忆配置系统

#### 9.1 记忆属性定义
```python
class MemoryAttribute(BaseModel):
    name: str
    type: Any
    default_or_value: Any
    description: str
    whether_embedding: bool = False
    embedding_template: Optional[str] = None
```

#### 9.2 配置生成
```python
def memory_config_societyagent(distributions, class_config=None):
    attributes = {
        "name": MemoryAttribute(name="name", type=str, 
                               default_or_value="unknown", whether_embedding=True),
        "emotion": MemoryAttribute(name="emotion", type=dict,
                                  default_or_value={"sadness": 5, "joy": 5, ...}),
        # ... 更多属性
    }
    return MemoryConfig(attributes=attributes)
```

### 10. 工作流程总结

1. **初始化**: 根据配置创建记忆结构，初始化向量存储
2. **插入**: 新信息通过KVMemory或StreamMemory存储，需要嵌入的字段自动生成向量
3. **编码**: 使用FastEmbed生成稀疏向量嵌入，存储到Qdrant
4. **存储**: 双重存储机制（内存+向量数据库）
5. **检索**: 支持精确检索和语义相似性搜索
6. **更新**: 支持替换和合并两种更新模式
7. **遗忘**: 通过时间衰减和容量限制实现自然遗忘
8. **认知**: 定期进行态度和思维更新，形成长期记忆

### 11. 核心技术创新

- **稀疏向量存储**: 使用Qdrant和FastEmbed实现高效的语义搜索
- **记忆配置系统**: 灵活的配置机制支持不同类型的智能体
- **认知处理流程**: 自动化的态度更新和思维反思机制
- **多层次检索**: 支持精确匹配和语义相似性搜索
- **异步并发**: 全异步设计，支持高并发操作
- **自然遗忘**: 通过时间衰减和容量限制模拟人类记忆特性

---

## 6. API 接口

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

#### 记忆互通与语义整合示例

```python
# 1. 跨topic的语义搜索（记忆互通）
# 搜索所有与"购物"相关的记忆，不限制topic
shopping_experiences = await memory.stream.search(
    query="shopping experience",
    top_k=10
)

# 2. 特定topic的过滤搜索
social_shopping = await memory.stream.search(
    query="shopping with friends",
    topic="social",
    top_k=5
)

# 3. 跨行为类型的记忆关联
# 创建认知记忆，关联多个topic的记忆
cognition_id = await memory.stream.add(
    topic="cognition",
    description="I had a great shopping day with friends at the mall"
)

# 关联相关的社交、移动、经济记忆
await memory.stream.add_cognition_to_memory(
    memory_ids=[social_id, mobility_id, economy_id],
    cognition=cognition_id
)

# 4. 基于综合记忆的决策
# 搜索所有相关记忆做决策
all_related_memories = await memory.stream.search(
    query="mall shopping experience",
    day_range=(current_day-30, current_day)
)

# 基于跨topic的记忆做决策
if "positive" in all_related_memories:
    decision = "I'll go to the mall because I had good experiences there"
else:
    decision = "I'll try a different shopping location"
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

## 7. 使用示例（Usage Examples）

### 1. 创建智能体记忆系统（Creating Agent Memory System）

```python
# 1. 创建记忆配置（Create Memory Configuration）
memory_config = memory_config_societyagent(distributions)

# 2. 创建记忆实例（Create Memory Instance）
memory = Memory(
    environment=environment,
    embedding=embedding,
    memory_config=memory_config
)

# 3. 初始化向量化（Initialize Embeddings）
await memory.initialize_embeddings()

# 4. 创建智能体（Create Agent）
agent = SocietyAgent(
    id=1,
    name="Alice",
    toolbox=toolbox,
    memory=memory
)
```

### 2. 智能体行为与记忆交互（Agent Behavior & Memory Interaction）

```python
# 智能体执行行为（Agent Behavior Execution）
async def agent_behavior():
    # 获取当前需求（Get Current Needs）
    hunger = await agent.memory.status.get("hunger_satisfaction")
    
    if hunger < 0.3:
        # 记录决策（Record Decision）
        await agent.memory.stream.add(
            topic="decision",
            description="I decided to eat because I'm hungry"
        )
        
        # 执行吃饭行为（Execute Eating Behavior）
        result = await agent.eat()
        
        # 更新状态（Update Status）
        await agent.memory.status.update("hunger_satisfaction", 0.9)
        
        # 记录结果（Record Result）
        await agent.memory.stream.add(
            topic="activity",
            description=f"I ate and now feel satisfied: {result}"
        )
```

### 3. 记忆搜索与决策（Memory Search & Decision Making）

```python
# 基于记忆做决策（Make Decisions Based on Memory）
async def make_decision():
    # 搜索相关经验（Search Related Experiences）
    past_experiences = await agent.memory.stream.search(
        query="shopping at this store",
        top_k=5
    )
    
    # 搜索情绪状态（Search Emotional State）
    happy_memories = await agent.memory.status.search(
        query="happy shopping",
        top_k=3
    )
    
    # 基于记忆做决策（Make Decision Based on Memory）
    if "positive" in happy_memories:
        decision = "I'll go shopping because I had good experiences before"
    else:
        decision = "I'm hesitant about shopping due to past experiences"
    
    # 记录决策过程（Record Decision Process）
    await agent.memory.stream.add(
        topic="cognition",
        description=f"Decision: {decision}. Based on: {past_experiences}"
    )
```

---

## 8. 扩展性设计

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

## 9. 案例研究

### 案例1：飓风影响模拟（Hurricane Impact Simulation）

#### 场景描述
模拟飓风对城市居民的影响，研究居民在灾难中的行为模式和记忆变化。

#### 记忆系统配置
```python
# 智能体配置（Agent Configuration）
AgentConfig(
    agent_class="citizen",
    number=100,
    memory_from_file="profiles_hurricane.json",
)

# 记忆属性（Memory Attributes）
{
    "home": 500020313,           # 家庭位置
    "work": 500016351,           # 工作位置
    "gender": "female",          # 性别
    "race": "white",             # 种族
    "education": "below bachelor", # 教育水平
    "income": 78231,             # 收入
    "consumption": "slightly high", # 消费水平
    "age": 70,                   # 年龄
    "id": 1                      # 智能体ID
}
```

#### 记忆工作流程（Memory Workflow）

**1. 灾难前状态记忆（Pre-Disaster Status Memory）**
```python
# 存储正常生活状态
await memory.status.update("safety_satisfaction", 0.9)
await memory.status.update("emotion", {"joy": 7, "fear": 2, "sadness": 1})
await memory.status.update("current_need", "social")
```

**2. 灾难事件记忆（Disaster Event Memory）**
```python
# 记录飓风来临
await memory.stream.add(
    topic="disaster", 
    description="飓风警报响起，需要立即撤离"
)

# 记录撤离过程
await memory.stream.add(
    topic="evacuation", 
    description="前往避难所，路上遇到交通堵塞"
)
```

**3. 认知更新（Cognitive Update）**
```python
# 智能体反思灾难经历
await memory.stream.add(
    topic="cognition", 
    description="这次飓风让我意识到安全的重要性，需要更好的应急准备"
)

# 更新对安全的态度
await memory.status.update("attitude", {
    "emergency_preparedness": 9,  # 从3提升到9
    "government_trust": 6         # 从8降低到6
})
```

**4. 记忆搜索与决策（Memory Search & Decision）**
```python
# 搜索相关经历
past_disasters = await memory.stream.search(
    query="飓风 撤离 安全",
    topic="disaster",
    day_range=(current_day-30, current_day)
)

# 基于记忆做决策
if "positive" in past_disasters:
    decision = "相信政府指导，按计划撤离"
else:
    decision = "自主选择更安全的撤离路线"
```

#### 关键洞察（Key Insights）
- **情绪变化**：飓风期间恐惧情绪显著上升，安全需求成为主导
- **态度转变**：对应急准备的重视程度大幅提升
- **行为模式**：基于过往经历调整应对策略

---

### 案例2：社会极化研究（Social Polarization Study）

#### 场景描述
研究社交媒体信息如何影响用户的政治态度，模拟信息茧房（Echo Chamber）效应。

#### 记忆系统配置
```python
# 极化智能体配置
{
    "id": 1,
    "attitude": {
        "Whether to support stronger gun control?": 3  # 初始态度：反对
    },
    "social_network": ["agent_2", "agent_3"],  # 社交网络
    "information_sources": ["conservative_media"]  # 信息来源
}
```

#### 记忆工作流程（Memory Workflow）

**1. 初始态度记忆（Initial Attitude Memory）**
```python
# 存储初始政治态度
await memory.status.update("attitude", {
    "gun_control": 3,  # 1-10量表，3表示反对
    "immigration": 4,
    "climate_change": 2
})
```

**2. 信息接触记忆（Information Exposure Memory）**
```python
# 记录接触的信息
await memory.stream.add(
    topic="information", 
    description="阅读了关于枪支管制的新研究，显示管制措施效果有限"
)

# 记录社交互动
await memory.stream.add(
    topic="social", 
    description="与朋友Bob讨论了枪支管制问题，我们都认为现有法律已经足够"
)
```

**3. 态度更新过程（Attitude Update Process）**
```python
# 认知块处理态度更新
async def attitude_update(self):
    # 搜索相关信息
    gun_control_info = await memory.stream.search(
        query="枪支管制 gun control",
        top_k=10
    )
    
    # 基于信息更新态度
    new_attitude = await self.process_attitude_update(
        topic="gun_control",
        information=gun_control_info,
        current_attitude=3
    )
    
    # 更新记忆
    await memory.status.update("attitude", {
        "gun_control": new_attitude  # 可能从3变为2（更反对）
    })
```

**4. 信息茧房效应（Echo Chamber Effect）**
```python
# 搜索同质化信息
similar_views = await memory.stream.search(
    query="枪支管制 反对 问题",
    topic="information",
    day_range=(current_day-7, current_day)
)

# 强化现有态度
if len(similar_views) > 5:
    # 信息茧房效应：更多同质化信息强化现有态度
    attitude_strength += 0.1
```

#### 关键洞察（Key Insights）
- **态度极化**：接触同质化信息导致态度更加极端
- **信息茧房**：算法推荐加剧了观点分化
- **社交影响**：朋友的观点对态度变化有显著影响

---

### 案例3：UBI政策模拟（Universal Basic Income Simulation）

#### 场景描述
模拟全民基本收入（UBI）政策对不同收入群体的影响，研究政策对工作动机和消费行为的影响。

#### 记忆系统配置
```python
# 不同收入群体配置
high_income_agent = {
    "income": 100000,
    "work_skill": 0.9,
    "work_propensity": 0.8,
    "consumption_propensity": 0.6
}

low_income_agent = {
    "income": 25000,
    "work_skill": 0.3,
    "work_propensity": 0.9,
    "consumption_propensity": 0.9
}
```

#### 记忆工作流程（Memory Workflow）

**1. 政策实施前（Pre-Policy Implementation）**
```python
# 记录当前工作状态
await memory.stream.add(
    topic="work", 
    description="每天工作8小时，月收入5000元，生活压力较大"
)

# 记录消费行为
await memory.stream.add(
    topic="consumption", 
    description="主要购买生活必需品，很少进行娱乐消费"
)
```

**2. UBI政策实施（UBI Policy Implementation）**
```python
# 记录政策信息
await memory.stream.add(
    topic="policy", 
    description="政府宣布实施全民基本收入，每月额外获得2000元"
)

# 更新收入状态
await memory.status.update("income", 7000)  # 5000 + 2000
await memory.status.update("ubi_benefit", 2000)
```

**3. 行为变化记录（Behavior Change Recording）**
```python
# 记录工作决策
await memory.stream.add(
    topic="work_decision", 
    description="由于有了基本收入保障，决定减少工作时间，更多陪伴家人"
)

# 记录消费变化
await memory.stream.add(
    topic="consumption", 
    description="开始购买一些非必需品，如书籍和娱乐用品"
)
```

**4. 长期影响评估（Long-term Impact Assessment）**
```python
# 搜索工作相关记忆
work_memories = await memory.stream.search(
    query="工作 work 收入 income",
    topic="work",
    day_range=(policy_start_day, current_day)
)

# 搜索消费相关记忆
consumption_memories = await memory.stream.search(
    query="消费 consumption 购买 purchase",
    topic="consumption",
    day_range=(policy_start_day, current_day)
)

# 分析行为变化趋势
work_hours_trend = analyze_trend(work_memories)
consumption_trend = analyze_trend(consumption_memories)
```

#### 关键洞察（Key Insights）
- **工作动机**：高收入群体工作动机略有下降，低收入群体变化不大
- **消费行为**：所有群体的消费都有所增加，特别是非必需品
- **生活质量**：整体生活满意度和安全感显著提升
- **社会影响**：减少了收入不平等带来的社会压力

---

### 案例总结（Case Study Summary）

这三个案例展示了AgentSociety记忆系统在不同场景下的应用：

1. **灾难模拟**：展示了记忆系统如何记录和影响危机应对行为
2. **社会研究**：展示了记忆系统如何模拟复杂的社会心理过程
3. **政策评估**：展示了记忆系统如何评估政策对个体行为的影响

每个案例都体现了记忆系统的核心价值：
- **状态记忆**记录智能体的当前状态和属性
- **事件记忆**记录时间序列的行为和经历
- **认知记忆**记录智能体的思考和态度变化
- **智能搜索**帮助智能体基于历史经验做决策