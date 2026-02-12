# 行为树设计规范

## 一、设计原则

### 1.1 逻辑在节点，结构在配置

**核心思想**：叶子节点封装原子逻辑，行为树配置只负责组合。

```
┌─────────────────────────────────────────────┐
│              行为树配置 (JSON)               │
│  - 定义节点的组合方式                         │
│  - 纯数据，无逻辑                            │
│  - 随时可修改，无需编译                       │
└─────────────────────────────────────────────┘
                      │
                      │ 引用
                      ▼
┌─────────────────────────────────────────────┐
│              叶子节点 (Go 代码)              │
│  - 封装原子化逻辑                            │
│  - 高度复用                                  │
│  - 一次实现，处处可用                         │
└─────────────────────────────────────────────┘
```

### 1.2 单一职责（高内聚）

**每个叶子节点只做一件事**。

```
✅ 好的设计：
   StopMove      → 只停止移动
   ClearPath     → 只清除路径
   SetTargetType → 只设置目标类型

❌ 差的设计：
   StopAndClear  → 停止 + 清除 + 重置（职责混杂）
```

### 1.3 无状态依赖（低耦合）

**叶子节点不知道自己在哪棵树、被谁调用**。

```
✅ 好的设计：
   StopMove 节点只关心"停止当前实体的移动"
   不关心是在 patrol 树还是 pursuit 树中

❌ 差的设计：
   节点内部判断"如果当前是追捕状态，则..."
```

### 1.4 参数化配置（可配置）

**通过参数控制节点行为，而不是写多个相似节点**。

```
✅ 好的设计：
   SetPathFindType(type: "NavMesh")
   SetPathFindType(type: "RoadNetwork")
   SetPathFindType(type: "None")

❌ 差的设计：
   SetNavMeshPathFind
   SetRoadNetworkPathFind
   ClearPathFindType
```

### 1.5 命名即文档（可读性）

**看节点名就知道做什么**。

```
✅ 好的命名：
   StopMove              → 停止移动
   SetTargetEntity       → 设置目标实体
   QueryRoadNetworkPath  → 查询路网路径

❌ 差的命名：
   DoAction1
   Helper
   Process
```

### 1.6 节点颗粒度（分层抽象）

**问题**：原子节点太细，配置复杂；复合节点太粗，灵活性差。

**解决方案**：两层节点体系。

```
┌─────────────────────────────────────────────────────────┐
│  业务动作节点 (Action Node) - 策划使用                    │
│  ─────────────────────────────────────────────────────  │
│  StartPursuit, StartScheduleMove, EnterIdle, EnterDialog│
│  - 一个节点 = 一个完整的业务动作                          │
│  - 参数使用业务语言，无需理解技术细节                      │
│  - 配置简单，10行以内完成一棵树                           │
└─────────────────────────────────────────────────────────┘
                         │
                         │ 内部封装
                         ▼
┌─────────────────────────────────────────────────────────┐
│  原子操作节点 (Primitive Node) - 程序员使用               │
│  ─────────────────────────────────────────────────────  │
│  SetFeature, SyncFeatureToBlackboard, SetPathFindType   │
│  - 细粒度控制，单一职责                                   │
│  - 需要理解特征值、黑板、组件等技术概念                    │
│  - 用于实现业务动作节点，或处理特殊需求                    │
└─────────────────────────────────────────────────────────┘
```

**示例对比**：

```
❌ 原子节点配置（80行，策划难以理解）：
   Sequence
   ├── SyncFeatureToBlackboard (mappings: {feature_pursuit_entity_id: player_id})
   ├── ClearPath
   ├── StartRun
   ├── SetPathFindType (type: NavMesh)
   ├── SetTargetEntity (entity_id_key: player_id)
   └── Selector
       ├── Sequence [CheckCondition + SetTargetType(Player)]
       └── SetTargetType (WayPoint)

✅ 业务动作节点配置（10行，策划易理解）：
   StartPursuit (target_feature: feature_pursuit_entity_id)
```

**颗粒度判断标准**：

| 问题 | 答案 | 建议 |
|------|------|------|
| 策划能理解这个节点吗？ | 否 | 封装为业务动作节点 |
| 这个操作是否经常组合出现？ | 是 | 封装为业务动作节点 |
| 需要在不同场景灵活组合吗？ | 是 | 保持原子节点 |
| 参数是否涉及技术细节？ | 是 | 封装并简化参数 |

### 1.7 架构边界（与 GSS Brain 的职责划分）

**问题**：警察巡逻 → 发现坏人追击 → 丢失目标调查 → 返回巡逻，这是一棵行为树还是多棵？

**答案**：**多棵行为树**，由 GSS Brain 负责高层状态切换。

```
┌─────────────────────────────────────────────────────────────┐
│                    GSS Brain (高层决策)                      │
│              负责：状态切换、条件判断、Plan 选择               │
│                                                             │
│  patrol ──(发现坏人)──> pursuit ──(丢失目标)──> investigate  │
│    ↑                                              │         │
│    └──────────────(调查无果)──────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ 产生 Plan
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    行为树 (低层执行)                          │
│                负责：单个 Plan 内的具体行为控制                │
│                                                             │
│  patrol_tree:     按路点移动 → 到达后等待 → 下一个路点        │
│  pursuit_tree:    追逐目标 → 保持跟踪                        │
│  investigate_tree: 移动到位置 → 环顾四周 → 搜索              │
└─────────────────────────────────────────────────────────────┘
```

**边界划分原则**：

| 层级 | 负责内容 | 示例 | 决策类型 |
|------|----------|------|----------|
| **GSS Brain** | 战略决策、状态切换 | "要追人" vs "要巡逻" | 做什么 (What) |
| **行为树** | 战术执行、行为细节 | "怎么追"、"怎么巡逻" | 怎么做 (How) |
| **叶子节点** | 原子动作 | 移动、转向、等待 | 具体动作 (Action) |

**两种方案对比**：

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **多棵简单树 + Brain** | 每个 Plan 一棵行为树，Brain 管理切换 | 职责清晰、易维护、可接入 AI/LLM | 两套系统 |
| **一棵复杂树** | 所有逻辑在一棵大树中 | 可视化集中 | 树太复杂、难以维护、不利于 AI 接入 |

**当前架构选择**：方案 A（多棵简单树 + Brain）

**实际例子：警察 NPC**

```
GSS Brain 规则:
┌────────────────────────────────────────────────┐
│ IF feature_enemy_in_sight THEN Plan = pursuit  │
│ IF feature_lost_target THEN Plan = investigate │
│ IF feature_investigate_done THEN Plan = patrol │
│ DEFAULT Plan = patrol                          │
└────────────────────────────────────────────────┘

行为树配置:
├── patrol_tree.json       # 按路点巡逻的具体行为
├── pursuit_tree.json      # 追击的具体行为
└── investigate_tree.json  # 调查的具体行为
```

**设计决策指南**：

| 场景 | 放在哪里 | 原因 |
|------|----------|------|
| "是否要追击敌人" | GSS Brain | 战略决策，涉及状态切换 |
| "追击时走哪条路" | 行为树 | 战术细节，不影响状态 |
| "追击时遇到障碍物绕行" | 行为树 | 行为内部的条件分支 |
| "追击时敌人消失了" | GSS Brain | 触发状态切换（pursuit → investigate） |
| "调查时在原地转圈看" | 行为树 | 调查行为的具体实现 |

### 1.8 行为树适用场景

**问题**：是不是只有需要 tick 处理的行为，才需要行为树？

**答案**：**不完全是**。行为树的价值不仅在于 tick 处理，还在于组合能力和可配置性。

**场景分类**：

| 场景 | 是否需要 tick | 是否适合用行为树 | 原因 |
|------|--------------|-----------------|------|
| 持续移动到目标点 | ✅ 需要 | ✅ 适合 | 需要每帧检查是否到达 |
| 等待一段时间 | ✅ 需要 | ✅ 适合 | 需要累计时间判断 |
| 追逐玩家 | ✅ 需要 | ✅ 适合 | 需要持续更新目标位置 |
| 进入对话状态 | ❌ 立即完成 | ✅ 适合 | 需要执行序列操作（清除特征→设置暂停→设置状态） |
| 停止移动 | ❌ 立即完成 | ✅ 适合 | 需要 entry/exit 生命周期管理 |
| 播放动画 | ❌ 立即触发 | ⚠️ 可选 | 简单动作可直接调用，复杂序列用 BT |

**同步多步骤 vs 异步多步骤**：

同步多步骤（一帧内完成）：
```json
{
  "type": "Sequence",
  "children": [
    { "type": "Log", "params": { "message": "step1" } },
    { "type": "EnterDialog" },
    { "type": "Log", "params": { "message": "step3" } }
  ]
}
```
这三个节点都是 `OnEnter` 直接返回 `Success`，**一帧内全部执行完**。

异步多步骤（跨多帧）：
```json
{
  "type": "Sequence",
  "children": [
    { "type": "MoveTo", "params": { "x": 100, "y": 200 } },
    { "type": "Wait", "params": { "duration_ms": 3000 } },
    { "type": "PlayAnimation", "params": { "name": "collect" } }
  ]
}
```
`MoveTo` 和 `Wait` 返回 `Running`，**需要多帧 tick** 才能完成。

**简单判断标准**：

| 情况 | 推荐方案 |
|------|---------|
| 单一同步操作 | 可以直接调用函数，不用 BT |
| 多个同步操作（固定顺序） | 可以封装成一个函数，或用 BT |
| 多个同步操作（策划需要调整） | 用 BT，方便配置 |
| 包含任何异步操作 | 必须用 BT 或类似的 tick 机制 |

**行为树的核心价值**：

1. **生命周期管理** - `on_entry`/`root`/`on_exit` 三阶段，确保状态清理
2. **组合能力** - Sequence/Selector 可以组合多个操作
3. **可配置性** - 策划可以通过 JSON 调整行为，无需改代码
4. **调试友好** - Log 节点、统一的执行流程便于排查

**结论**：需要 tick 的行为必须用行为树；不需要 tick 但包含多步骤的行为推荐用行为树（利用组合和配置化优势）。

---

## 二、核心概念

### 2.1 节点类型

| 类型 | 职责 | 示例 |
|------|------|------|
| **控制节点** | 控制子节点的执行流程 | Sequence, Selector, Parallel |
| **装饰节点** | 修改子节点的行为 | Inverter, Repeat, Timeout |
| **叶子节点** | 执行具体逻辑 | StopMove, SetFeature, Log |

### 2.2 叶子节点分类

| 分类 | 职责 | 示例 |
|------|------|------|
| **Action** | 执行动作，改变世界状态 | StopMove, StartRun, SetFeature |
| **Condition** | 检查条件，不改变状态 | CheckCondition, HasTarget |
| **Query** | 查询数据，写入黑板 | GetScheduleData, FindNearestPoint |

### 2.3 行为树结构

```json
{
  "name": "树的唯一标识",
  "root": {
    "type": "控制节点类型",
    "children": [
      { "type": "叶子节点", "params": { ... } },
      { "type": "叶子节点", "params": { ... } }
    ]
  }
}
```

**配置只关心**：用什么节点、怎么组合、传什么参数。

### 2.4 配置字段说明

**配置结构定义** (`config/types.go`):

```go
// BTreeConfig 行为树根配置
type BTreeConfig struct {
    Name        string     `json:"name"`        // 树的唯一标识
    Description string     `json:"description"` // 描述（可选）
    Root        NodeConfig `json:"root"`        // 根节点
}

// NodeConfig 节点配置（递归结构）
type NodeConfig struct {
    Type     string         `json:"type"`     // 节点类型 → 决定创建哪种节点
    Params   map[string]any `json:"params"`   // 节点参数 → 传递给节点构造函数
    Children []NodeConfig   `json:"children"` // 子节点列表（控制节点用）
    Child    *NodeConfig    `json:"child"`    // 单子节点（装饰节点用）
}
```

**字段作用**：

| 字段 | 作用 | 使用者 |
|------|------|--------|
| `type` | 节点类型名，查找对应的创建函数 | `factory.Create()` |
| `params` | 节点参数，传递给节点构造函数 | 各 `createXxxNode()` |
| `children` | 控制节点的子节点列表 | `loader.buildNode()` |
| `child` | 装饰节点的单个子节点 | `loader.buildNode()` |

### 2.5 配置解析流程

```
JSON 配置文件 (pursuit_entry.json)
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. BTreeLoader.LoadFromJSON(data)           [config/loader.go]
│    - json.Unmarshal → BTreeConfig 结构                       │
│    - 调用 buildNode(&cfg.Root)                               │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. BTreeLoader.buildNode(cfg *NodeConfig)   [config/loader.go]
│    - 调用 factory.Create(cfg) 创建当前节点                   │
│    - 如果有 children，递归调用 buildNode 创建子节点           │
│    - 将子节点添加到父节点：controlNode.AddChild(child)       │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. NodeFactory.Create(cfg *NodeConfig)      [nodes/factory.go]
│    - 根据 cfg.Type 查找创建函数：creators["Log"]             │
│    - 调用创建函数：createLogNode(cfg)                        │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. createLogNode(cfg *NodeConfig)           [nodes/factory.go]
│    - 从 cfg.Params 读取参数：                                │
│        message, _ := cfg.GetParamString("message")          │
│        level, _ := cfg.GetParamString("level")              │
│    - 创建节点实例：return NewLogNode(message, level)         │
└─────────────────────────────────────────────────────────────┘
```

**具体示例**：

JSON 配置：
```json
{
  "type": "Log",
  "params": {
    "message": "[PursuitEntry] start",
    "level": "debug"
  }
}
```

对应的创建函数：
```go
func createLogNode(cfg *config.NodeConfig) (node.IBtNode, error) {
    // 从 params 读取参数
    message, ok := cfg.GetParamString("message")
    if !ok {
        message = "LogNode"  // 默认值
    }
    level, _ := cfg.GetParamString("level")

    // 创建节点实例
    return NewLogNodeWithLevel(message, LogLevel(level)), nil
}
```

**参数读取方法** (`config/types.go`):

```go
cfg.GetParamString("message")   // → (string, bool)
cfg.GetParamInt64("duration")   // → (int64, bool)
cfg.GetParamFloat32("speed")    // → (float32, bool)
cfg.GetParamBool("paused")      // → (bool, bool)
cfg.GetParamAny("target")       // → (any, bool) 用于复杂类型如 []float32
```

**控制节点的 children 处理**：

```json
{
  "type": "Sequence",
  "children": [
    { "type": "Log", "params": { "message": "A" } },
    { "type": "StartPursuit" }
  ]
}
```

Loader 递归处理：
```go
func (l *BTreeLoader) buildNode(cfg *NodeConfig) (node.IBtNode, error) {
    // 1. 创建节点本身
    n, _ := l.factory.Create(cfg)  // 创建 Sequence

    // 2. 递归处理 children
    if len(cfg.Children) > 0 {
        controlNode := n.(ControlNode)
        for _, childCfg := range cfg.Children {
            child, _ := l.buildNode(&childCfg)  // 递归
            controlNode.AddChild(child)
        }
    }
    return n, nil
}
```

**组件职责总结**：

| 文件 | 职责 |
|------|------|
| `config/types.go` | 定义配置结构 + 参数读取方法 |
| `config/loader.go` | 解析 JSON + 递归构建节点树 |
| `nodes/factory.go` | 根据 `type` 创建节点 + 处理 `params` |
| `trees/*.json` | 定义"用什么节点、怎么组合、传什么参数" |

---

## 三、叶子节点设计

### 3.1 节点接口

```go
type IBtNode interface {
    OnEnter(ctx *BtContext) BtNodeStatus  // 进入时
    OnTick(ctx *BtContext) BtNodeStatus   // 每帧执行
    OnExit(ctx *BtContext)                // 退出时清理
}
```

### 3.2 节点实现示例

```go
// StopMove - 停止移动
type StopMoveNode struct {
    BaseNode
}

func (n *StopMoveNode) OnEnter(ctx *BtContext) BtNodeStatus {
    moveComp := GetMoveComponent(ctx.EntityID)
    moveComp.Stop()
    return BtNodeStatusSuccess
}

// SetPathFindType - 设置寻路类型（参数化）
type SetPathFindTypeNode struct {
    BaseNode
    Type string `json:"type"` // NavMesh, RoadNetwork, None
}

func (n *SetPathFindTypeNode) OnEnter(ctx *BtContext) BtNodeStatus {
    moveComp := GetMoveComponent(ctx.EntityID)
    moveComp.SetPathFindType(n.Type)
    return BtNodeStatusSuccess
}
```

### 3.3 节点注册（带元数据）

```go
// factory.go - 使用 RegisterWithMeta 注册节点和元数据
f.RegisterWithMeta(&NodeMeta{
    Type:        "MoveTo",
    Category:    CategoryMovement,
    Description: "移动到指定位置",
    Params: []ParamMeta{
        {Name: "target_key", Type: ParamTypeString, Description: "从黑板读取目标位置"},
        {Name: "target", Type: ParamTypeVec3, Description: "直接指定目标坐标 [x,y,z]"},
        {Name: "speed", Type: ParamTypeFloat32, Description: "移动速度"},
    },
    Example: `{"type": "MoveTo", "params": {"target_key": "patrol_point"}}`,
}, createMoveToNode)
```

### 3.4 节点分类

| 分类 | 常量 | 说明 |
|------|------|------|
| `control` | CategoryControl | 控制节点（Sequence, Selector） |
| `movement` | CategoryMovement | 移动类节点 |
| `path` | CategoryPath | 路径/导航节点 |
| `feature` | CategoryFeature | 特征值节点 |
| `blackboard` | CategoryBlackboard | 黑板操作节点 |
| `dialog` | CategoryDialog | 对话类节点 |
| `schedule` | CategorySchedule | 日程节点 |
| `debug` | CategoryDebug | 调试节点 |
| `specific` | CategorySpecific | 特定业务节点 |

---

## 四、业务动作节点（策划用）

业务动作节点封装完整的业务逻辑，策划只需配置业务参数。

### 4.1 规划中的业务动作节点

| 节点 | 用途 | 参数 | 封装的原子操作 |
|------|------|------|---------------|
| `StartPursuit` | 开始追逐玩家 | target_feature | ClearPath → StartRun → SetPathFindType(NavMesh) → SetTargetEntity → SetTargetType |
| `StopPursuit` | 停止追逐 | - | StopMove → SetPathFindType(None) → SetTargetEntity(0) → SetTargetType(None) |
| `StartScheduleMove` | 按日程移动 | - | 读日程 → QueryRoadNetworkPath → StartMove → SetPathFindType → SetPointList |
| `EnterIdle` | 进入空闲状态 | - | GetScheduleData → SetDialogOutFinishStamp → SetTownNpcOutDuration → SetTransformFromFeature |
| `EnterDialog` | 进入对话 | - | ClearDialogEventFeature → SetOutPause → SetDialogState → SetDialogEventType |
| `ExitDialog` | 退出对话 | - | ClearDialogEventFeature → 计算对话时长 → UpdateOutFinishStamp → SetDialogState |
| `EnterHomeIdle` | 进入家中空闲 | - | SetFeature(out_timeout) → SetTransformFromFeature |
| `StartInvestigate` | 开始调查 | target_feature | SetupNavMeshPathToFeaturePos |

### 4.2 业务动作节点配置示例

**追逐行为配置**：
```json
{
  "name": "pursuit_entry",
  "root": {
    "type": "StartPursuit",
    "params": {
      "target_feature": "feature_pursuit_entity_id"
    }
  }
}
```

**空闲行为配置**：
```json
{
  "name": "idle_entry",
  "root": {
    "type": "EnterIdle"
  }
}
```

### 4.3 业务动作节点实现模板

```go
// StartPursuitNode 开始追逐（业务动作节点）
type StartPursuitNode struct {
    node.BaseNode
    TargetFeature string // 目标实体ID的特征key
}

func (n *StartPursuitNode) OnEnter(ctx *context.BtContext) node.BtNodeStatus {
    // 1. 从特征值读取目标实体ID
    targetID := ctx.GetFeatureUint64(n.TargetFeature)

    // 2. 获取移动组件
    moveComp := getMoveComponent(ctx.EntityID)

    // 3. 执行完整的追逐初始化
    moveComp.Clear()
    moveComp.StartRun()
    moveComp.SetPathFindType(PathFindType_NavMesh)
    moveComp.SetTargetEntity(targetID)
    if targetID != 0 {
        moveComp.SetTargetType(TargetType_Player)
    } else {
        moveComp.SetTargetType(TargetType_WayPoint)
    }

    return node.BtNodeStatusSuccess
}
```

---

## 五、原子操作节点（程序员用）

原子操作节点提供细粒度控制，用于实现业务动作节点或处理特殊需求。

### 5.1 移动控制

| 节点 | 参数 | 职责 |
|------|------|------|
| `StopMove` | - | 停止移动 |
| `StartMove` | - | 开始移动 |
| `StartRun` | - | 开始奔跑 |
| `ClearPath` | - | 清除路径 |
| `SetPathFindType` | type: NavMesh/RoadNetwork/None | 设置寻路类型 |
| `SetTargetType` | type: Player/WayPoint/None | 设置目标类型 |
| `SetTargetEntity` | entity_id / entity_id_key | 设置目标实体 |

### 5.2 位置与变换

| 节点 | 参数 | 职责 |
|------|------|------|
| `SetTransformFromFeature` | pos_keys, rot_keys | 从特征值设置位置 |
| `MoveTo` | target_key / target_pos | 移动到目标点 |
| `FindNearestRoadPoint` | output_key | 查找最近路点 |

### 5.3 特征值操作

| 节点 | 参数 | 职责 |
|------|------|------|
| `SetFeature` | feature_key, feature_value, ttl_ms | 设置特征值 |
| `SyncFeatureToBlackboard` | mappings | 同步特征值到黑板 |
| `GetScheduleData` | output_keys | 获取日程数据 |

### 5.4 条件检查

| 节点 | 参数 | 职责 |
|------|------|------|
| `CheckCondition` | blackboard_key, operator, value | 检查黑板值 |
| `CheckFeature` | feature_key, operator, value | 检查特征值 |

### 5.5 通用

| 节点 | 参数 | 职责 |
|------|------|------|
| `Log` | message, level | 打印日志 |
| `Wait` | duration_ms / duration_key | 等待指定时间 |
| `GetCurrentTime` | output_key | 获取当前时间戳 |

### 5.6 节点检索与文档

**查看所有节点摘要**：
```bash
go test -v ./servers/scene_server/internal/common/ai/bt/nodes/... -run TestNodeRegistryPopulated
```

**搜索节点**：
```go
// 按关键词搜索（支持类型名和描述）
results := nodes.SearchNodes("Move")  // → MoveTo, StopMove, StartMove

// 按分类获取
nodes := nodes.GetMetaByCategory(CategoryDialog)  // → 所有对话节点

// 获取单个节点元数据
meta, ok := nodes.GetMeta("MoveTo")
```

**自动生成文档**：
```bash
# 生成 NODE_REFERENCE.md 和 btree_schema.json
WRITE_NODE_DOC=1 go test ./servers/scene_server/internal/common/ai/bt/nodes/... -run WriteNodeDocToFile
```

生成的文件：
- `bt/trees/NODE_REFERENCE.md` - Markdown 格式的节点参考手册
- `bt/trees/btree_schema.json` - JSON Schema（IDE 补全用）

---

## 六、行为树配置示例

### 5.1 简单行为：空闲

```json
{
  "name": "idle",
  "root": {
    "type": "Sequence",
    "children": [
      { "type": "StopMove" },
      { "type": "SetTransformFromFeature", "params": {
          "pos_keys": ["feature_posx", "feature_posy", "feature_posz"],
          "rot_keys": ["feature_rotx", "feature_roty", "feature_rotz"]
      }},
      { "type": "Log", "params": { "message": "[Idle] waiting", "level": "debug" }}
    ]
  }
}
```

### 5.2 复杂行为：追捕

```json
{
  "name": "pursuit",
  "root": {
    "type": "Sequence",
    "children": [
      { "type": "SyncFeatureToBlackboard", "params": {
          "mappings": { "feature_pursuit_entity_id": "target_id" }
      }},
      { "type": "ClearPath" },
      { "type": "StartRun" },
      { "type": "SetPathFindType", "params": { "type": "NavMesh" }},
      { "type": "SetTargetEntity", "params": { "entity_id_key": "target_id" }},
      { "type": "Selector", "children": [
          { "type": "Sequence", "children": [
              { "type": "CheckCondition", "params": {
                  "blackboard_key": "target_id", "operator": "!=", "value": 0
              }},
              { "type": "SetTargetType", "params": { "type": "Player" }}
          ]},
          { "type": "SetTargetType", "params": { "type": "WayPoint" }}
      ]}
    ]
  }
}
```

### 5.3 清理行为：重置移动状态

```json
{
  "name": "reset_move_state",
  "root": {
    "type": "Sequence",
    "children": [
      { "type": "StopMove" },
      { "type": "SetPathFindType", "params": { "type": "None" }},
      { "type": "SetTargetEntity", "params": { "entity_id": 0 }},
      { "type": "SetTargetType", "params": { "type": "None" }}
    ]
  }
}
```

---

## 七、Plan 配置

### 6.1 配置结构

```json
{
  "plan_name": {
    "tree": "主行为树",
    "entry_tree": "进入时执行的树（可选）",
    "exit_tree": "退出时执行的树（可选）"
  }
}
```

### 6.2 完整示例

```json
{
  "idle": {
    "tree": "idle",
    "entry_tree": null,
    "exit_tree": null
  },
  "move": {
    "tree": "move",
    "entry_tree": null,
    "exit_tree": "reset_move_state"
  },
  "pursuit": {
    "tree": "pursuit",
    "entry_tree": null,
    "exit_tree": "reset_move_state"
  },
  "dialog": {
    "tree": "dialog",
    "entry_tree": "stop_move_tree",
    "exit_tree": null
  }
}
```

### 6.3 执行流程

```
Plan A → Plan B 切换

1. 中断 Plan A 的主树（节点级 OnExit 清理）
2. 执行 Plan A 的 exit_tree（如果有）
3. 执行 Plan B 的 entry_tree（如果有）
4. 启动 Plan B 的主树
```

---

## 八、设计对比

### 7.1 为什么不用 SubTree

| SubTree 方式 | 叶子节点方式 |
|--------------|--------------|
| 树中套树，层级复杂 | 扁平结构，所见即所得 |
| 需要理解"子树"概念 | 只需理解"节点"概念 |
| 复用单元是"树" | 复用单元是"节点" |
| 配置 + 配置 | 代码 + 配置 |

### 7.2 职责分离

```
┌─────────────────────────────────────────────┐
│                  开发者                      │
│  实现叶子节点（Go 代码）                      │
│  - StopMove, SetFeature, CheckCondition     │
│  - 一次实现，永久复用                         │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│                  策划/配置                   │
│  编写行为树配置（JSON）                       │
│  - 组合现有节点                              │
│  - 调整参数                                  │
│  - 无需编程知识                              │
└─────────────────────────────────────────────┘
```

---

## 九、设计检查清单

### 添加新叶子节点时

| 检查项 | 说明 |
|--------|------|
| 单一职责 | 这个节点只做一件事吗？ |
| 参数化 | 能否通过参数支持多种场景，而不是写多个节点？ |
| 命名清晰 | 看名字能知道做什么吗？ |
| 无状态依赖 | 节点是否依赖"在哪棵树中"？（不应该）|
| **注册元数据** | 使用 `RegisterWithMeta` 并填写完整的 Category、Description、Params |
| **更新文档** | 运行 `WRITE_NODE_DOC=1 go test ...` 重新生成文档 |

### 编写行为树配置时

| 检查项 | 说明 |
|--------|------|
| 节点复用 | 是否使用了已有的标准节点？先用 `SearchNodes` 查找 |
| 结构清晰 | 树的结构是否容易理解？ |
| 参数正确 | 节点参数是否符合 `NODE_REFERENCE.md` 文档定义？ |

---

## 十、现有节点统计

**共 38 个节点**，按分类统计：

| 分类 | 数量 | 示例节点 |
|------|------|----------|
| 控制节点 | 2 | Sequence, Selector |
| 移动类 | 3 | MoveTo, StopMove, LookAt |
| 路径/导航 | 12 | SetPathFindType, QueryRoadNetworkPath, FindNearestRoadPoint |
| 特征值 | 4 | SetFeature, CheckCondition, SyncFeatureToBlackboard |
| 黑板操作 | 1 | SetBlackboard |
| 对话类 | 10 | PushDialogTask, SetDialogState, SetDialogPause |
| 日程 | 2 | GetScheduleData, GetScheduleKey |
| 调试 | 2 | Log, Wait |
| 特定业务 | 2 | SetInvestigatePlayer, SetSakuraControlEventType |

**完整节点列表**：运行测试或查看 `bt/trees/NODE_REFERENCE.md`

```bash
go test -v ./servers/scene_server/internal/common/ai/bt/nodes/... -run TestNodeRegistryPopulated
```

---

## 十一、目录结构

```
bt/
├── config/
│   ├── types.go              # BTreeConfig, NodeConfig 配置类型
│   ├── loader.go             # JSON 配置加载器
│   └── plan_config.go        # PlanConfig 加载器
│
├── context/
│   └── context.go            # BtContext 执行上下文
│
├── node/
│   └── interface.go          # IBtNode 接口定义
│
├── nodes/
│   ├── factory.go            # 节点注册工厂（含元数据）
│   ├── registry.go           # 节点元数据注册表、搜索、文档生成
│   ├── base.go               # BaseNode 基类
│   │
│   ├── # 控制节点
│   ├── sequence.go           # 顺序节点
│   ├── selector.go           # 选择节点
│   │
│   ├── # 通用叶子节点
│   ├── log.go                # 日志
│   ├── wait.go               # 等待
│   ├── check_condition.go    # 条件检查
│   ├── set_blackboard.go     # 黑板操作
│   │
│   ├── # 移动相关
│   ├── move_to.go
│   ├── stop_move.go
│   ├── look_at.go
│   ├── path_control.go       # SetPathFindType, SetTargetType 等
│   ├── navigation.go         # FindNearestRoadPoint 等
│   ├── roadnetwork.go        # QueryRoadNetworkPath 等
│   │
│   ├── # 特征值相关
│   ├── set_feature.go
│   ├── feature_sync.go       # SyncFeatureToBlackboard
│   │
│   ├── # 业务特有
│   ├── schedule.go           # 日程相关
│   ├── dialog.go             # 对话相关
│   ├── dialog_ext.go         # 对话扩展节点
│   └── specific_comp.go      # 特定组件节点
│
├── runner/
│   └── runner.go             # BtRunner 运行器
│
└── trees/
    ├── example_trees.go      # 代码定义的示例树 + 注册函数
    │
    ├── # Plan 配置（entry/main/exit 独立文件）
    ├── plan_config.json      # Plan 名称到行为树的映射
    ├── idle_entry.json
    ├── idle_main.json
    ├── idle_exit.json
    ├── move_entry.json
    ├── move_main.json
    ├── move_exit.json
    ├── ... (其他 Plan 的 entry/main/exit)
    │
    ├── # 自动生成的文档
    ├── NODE_REFERENCE.md     # 节点参考手册
    └── btree_schema.json     # JSON Schema（IDE 补全）
```
