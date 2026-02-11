---
name: dev-workflow
description: 软件工程全流程工作流。从需求文档出发，依次完成设计、审查、拆解、实现、验收、经验沉淀。支持多工程协作（业务工程、协议工程、配置工程）。
argument-hint: "[requirement-file] [--proto=相对路径] [--config=相对路径]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, Task, AskUserQuestion
---

你是一名软件工程流程编排专家。按照以下 Phase 1-7 共 7 个阶段有序推进，每个关键节点暂停等待用户确认后再继续。

---

## 辅助文档

本 skill 包含以下辅助文档，在执行相关阶段时应读取参考：

| 文档 | 用途 | 使用时机 |
|------|------|----------|
| `DB.md` | 数据库架构、存取流程、调试方法 | Phase 2 数据库设计、Phase 4 实现、**Phase 7 沉淀** |
| `PROTO.md` | 协议工程修改规范、客户端请求限制 | Phase 2 协议设计、Phase 4 实现、**Phase 7 沉淀** |
| `TEST.md` | 测试类型、测试规范、验证清单、压测机器人、容灾测试 | Phase 4 实现后测试、Phase 6 审查、上线前压测、**Phase 7 沉淀** |
| `REVIEW.md` | 代码审查规范、检查清单、容量限制、序列化检查 | Phase 6 产出审查、上线前检查、**Phase 7 沉淀** |
| `NPC.md` | NPC 系统架构、AI 决策、DB 存取流程、测试要点 | NPC 功能开发、Phase 4 实现、**Phase 7 沉淀** |

---

## 参数解析

从 $ARGUMENTS 中解析以下参数：

- **需求文档路径**：第一个参数（必须），如 `requirements/user-registration.md`
- **--proto=路径**：协议工程相对于当前业务工程的路径（可选），如 `--proto=../proto-repo`
- **--config=路径**：配置工程相对于当前业务工程的路径（可选），如 `--config=../config-repo`

### 默认路径

如果未指定 --proto 或 --config，使用以下默认路径：

| 工程 | 默认路径 | 说明 |
|------|----------|------|
| 协议工程 | `../proto/old_proto/` | Protocol Buffer 定义文件 |
| 配置工程 | `../config/RawTables/` | 游戏配置表（Excel/JSON）|

如果默认路径不存在或不适用，在 Phase 1 中向用户确认实际路径。

调用示例：
```
/dev-workflow requirements/feature.md --proto=../proto-repo --config=../config-repo
/dev-workflow requirements/feature.md --proto=../proto-repo
/dev-workflow requirements/feature.md   # 使用默认路径: proto=../proto/old_proto/, config=../config/RawTables/
```

---

## 工程关系模型

本工作流支持多工程协作，P1GoServer 项目的典型结构如下：

```
workspace/server/
├── P1GoServer/                ← 当前所在目录（业务工程，Go 版本）
│   ├── servers/                  各微服务代码
│   ├── common/                   共享库
│   ├── common/config/            游戏配置（从配置工程生成）
│   ├── resources/proto/          协议生成的代码（git submodule）
│   └── .claude/                  Claude Code 配置
│
├── server_old/                ← Rust 遗留业务工程 (../server_old/)
│   └── ...                       大部分逻辑已重构到 P1GoServer
│
├── proto/old_proto/           ← 默认协议工程 (../proto/old_proto/)
│   ├── scene/
│   │   └── xxx.proto             协议定义源文件
│   ├── logic/
│   └── module.proto              模块定义
│
└── config/RawTables/          ← 默认配置工程 (../config/RawTables/)
    ├── TownTask/                 任务配置表
    ├── TownNpc/                  NPC 配置表
    └── ...                       其他游戏配置表（Excel）
```

### Rust 遗留工程说明 (../server_old/)

`server_old` 是项目的 Rust 版本业务工程，目前处于**遗留状态**：

- **现状**：大部分业务逻辑已重构到当前的 Go 版本（P1GoServer）
- **用途**：
  - 理解历史实现逻辑（重构参考）
  - 查找尚未迁移的功能
  - 对比 Rust/Go 实现差异
- **注意**：新功能开发应在 P1GoServer 中进行，不要在 server_old 中新增代码

### 工程间关系

```
协议工程                         配置工程
(../proto/old_proto/)            (../config/RawTables/)
    │                                │
    │ 代码生成 (protoc)               │ 配置生成 (config_gen.exe)
    │                                │
    ▼                                ▼
        业务工程 (P1GoServer/)
        ├── resources/proto/     ← 协议生成产物
        └── common/config/       ← 配置生成产物 (cfg_*.go)
```

### 工程定位规则

- **业务工程**：当前工作目录（Claude Code 启动位置，即 P1GoServer/）
- **协议工程**：通过 --proto 参数指定，默认为 `../proto/old_proto/`
- **配置工程**：通过 --config 参数指定，默认为 `../config/RawTables/`

---

## Phase 1：需求解析与验证

### 1.1 读取需求文档

使用 Read 工具读取用户指定的需求文档路径，理解以下内容：

- **功能概述**：要实现什么功能
- **验收标准**：如何判定功能完成
- **依赖关系**：是否依赖其他功能或模块
- **涉及工程**：需要修改哪些工程（业务/协议/配置/Rust 遗留）
- **优先级**：功能重要性（用于资源分配）

### 1.2 工程定位

明确本次任务涉及的工程，确认各工程的实际路径：

1. **业务工程**：当前目录，无需确认
2. **协议工程**：
   - 如果指定了 `--proto`，验证路径是否存在
   - 如果使用默认路径 `../proto/old_proto/`，验证是否存在
   - 如果都不存在，询问用户实际的协议工程路径
3. **配置工程**：
   - 如果指定了 `--config`，验证路径是否存在
   - 如果使用默认路径 `../config/RawTables/`，验证是否存在
   - 如果都不存在，询问用户实际的配置工程路径
4. **Rust 遗留工程**：
   - 验证 `../server_old/` 是否存在
   - 如果不存在，标记为不可用

### 1.3 依赖检查

检查前置条件：

- **协议工程**：如果涉及协议修改，确认 protoc 编译环境是否就绪
- **配置工程**：如果涉及配置修改，确认配置生成工具是否就绪
- **数据库**：如果涉及数据库变更，确认数据库连接和权限
- **测试环境**：确认是否有可用的测试环境或本地 mock

### 1.4 输出摘要

输出以下信息等待用户确认：

```
需求摘要：
- 功能名称：[name]
- 验收标准：[acceptance criteria]
- 涉及工程：[business, proto, config, rust]
- 路径确认：
  * 业务工程：/path/to/P1GoServer/
  * 协议工程：/path/to/proto/
  * 配置工程：/path/to/config/
  * Rust 遗留工程：/path/to/server_old/ (可选)
- 依赖项：[dependencies]
- 风险点：[risks]
```

**等待用户确认后进入 Phase 2。**

---

## Phase 2：技术设计

### 2.1 架构设计

根据需求文档，设计整体架构：

1. **系统边界**：明确哪些功能在哪个工程中实现
2. **接口设计**：
   - 跨服务调用：设计 RPC 接口（协议工程）
   - 数据存储：设计 DB Schema（DB Migration）
   - 配置驱动：设计配置表结构（配置工程）
3. **状态流转**：设计核心业务的状态机（如有）
4. **错误处理**：定义错误码和异常场景

### 2.2 详细设计

分工程进行详细设计：

#### 业务工程设计 (P1GoServer/)

1. **模块划分**：
   - 新增哪些 service/manager/component
   - 修改哪些现有模块
   - 模块间的依赖关系

2. **数据结构**：
   - 新增/修改的 struct 定义
   - 序列化方式（JSON/Protobuf）
   - 持久化策略（内存/DB）

3. **接口定义**：
   - 对外暴露的 API（HTTP/RPC）
   - 内部组件间的调用接口
   - 回调接口设计（如有异步操作）

4. **并发控制**：
   - 需要加锁的共享资源
   - 锁的粒度和类型（读写锁/互斥锁）
   - 死锁风险评估

#### 协议工程设计 (../proto/)

1. **消息定义**：
   - 新增的 Request/Response 消息
   - 消息的字段定义和类型
   - 必选/可选字段说明

2. **枚举和常量**：
   - 新增的枚举类型
   - 常量定义（错误码、状态值等）

3. **字段设计原则**：
   - 遵循 Protobuf 最佳实践
   - 合理使用 repeated、oneof 等特性
   - 保留字段（reserved）用于未来扩展

#### 配置工程设计 (../config/)

1. **配置表设计**：
   - 需要新增/修改的配置表
   - 表字段定义（类型、默认值、约束）
   - 配置项之间的关系（外键/引用）

2. **配置加载**：
   - 配置生成器的修改点
   - 配置热更新策略（如需要）

#### 数据库设计

1. **表结构设计**：
   - 新增表的字段定义
   - 索引设计（查询优化）
   - 外键约束（数据一致性）

2. **Migration 脚本**：
   - DDL 语句（CREATE/ALTER TABLE）
   - 数据迁移（如有存量数据）
   - 回滚脚本（rollback）

#### Rust 遗留工程参考 (../server_old/)

1. **功能对标**：
   - 找出对应的 Rust 实现文件
   - 对比 Rust/Go 的实现差异
   - 提取可复用的设计思路

2. **兼容性考虑**：
   - 如果 Rust 工程仍在运行，需要考虑兼容性
   - 数据格式兼容（如数据库表结构）
   - 协议兼容（消息格式不变）

### 2.3 事务性设计

对于涉及多个状态变更的操作，设计事务边界和回滚逻辑：

1. **事务范围**：
   - 明确哪些操作必须在同一个事务中
   - 跨服务的事务处理（分布式事务/补偿机制）

2. **回滚机制**：
   - 每个操作失败后的回滚逻辑
   - 回滚的顺序（与执行相反）
   - 回滚失败的处理（二次回滚/告警）

3. **幂等性设计**：
   - 为每个操作设计幂等键
   - 幂等键的生成规则（用户ID+操作类型+时间戳等）
   - 幂等检查的实现位置

4. **并发控制**：
   - 需要加锁的关键路径
   - 锁的范围和持有时间
   - 死锁预防和检测

### 2.4 接口契约

定义跨工程协作的契约：

1. **协议工程 ↔ 业务工程**：
   - 消息格式和字段含义
   - 版本兼容性（字段可选/废弃）
   - 错误码映射

2. **配置工程 ↔ 业务工程**：
   - 配置项名称和类型
   - 配置更新的触发时机
   - 配置缺失的默认行为

3. **业务工程 ↔ 数据库**：
   - 读写分离策略（如有）
   - 事务隔离级别
   - 连接池配置

### 2.5 输出设计文档

生成完整的设计文档，包含：

1. **需求回顾**：简要回顾需求要点
2. **架构图**：系统架构和组件关系
3. **各工程详细设计**：
   - 业务工程：模块、接口、数据结构
   - 协议工程：消息定义、枚举、常量
   - 配置工程：配置表、加载逻辑
   - 数据库：表结构、索引、Migration
   - Rust 参考：对标的实现和差异
4. **事务性设计**：事务边界、回滚、幂等、并发控制
5. **接口契约**：各工程间的调用约定
6. **风险和缓解**：技术风险和应对方案

保存到 `docs/design-[feature-name].md`，**等待用户确认后进入 Phase 3。**

---

## Phase 3：任务拆解

### 3.1 任务拆解原则

将设计拆解为可独立执行的原子任务：

1. **单一职责**：每个任务只做一件事
2. **工程隔离**：不同工程的任务完全隔离（可并行）
3. **依赖明确**：任务间的依赖关系清晰
4. **可验证**：每个任务有明确的完成标准

### 3.2 任务分类

按工程分类任务：

#### 业务工程任务 (P1GoServer/)

- [ ] 基础设施：新增 struct、接口定义
- [ ] Service 层：实现业务逻辑
- [ ] Manager 层：状态管理和缓存
- [ ] Component 层（如 ECS）：组件定义
- [ ] 事件处理：事件订阅和发布
- [ ] 单元测试：核心逻辑测试（参考 `TEST.md`）
- [ ] 集成测试：跨模块交互测试（参考 `TEST.md`）

#### 协议工程任务 (../proto/)

- [ ] 新增消息定义
- [ ] 修改枚举和常量
- [ ] 运行 protoc 生成代码
- [ ] 更新 git submodule

#### 配置工程任务 (../config/)

- [ ] 新增配置表
- [ ] 修改配置生成器
- [ ] 运行配置生成工具
- [ ] 验证生成的代码

#### 数据库任务

- [ ] 编写 Migration 脚本（参考 `DB.md`）
- [ ] 本地测试 Migration（参考 `TEST.md` Migration 测试）
- [ ] 数据回滚脚本

#### Rust 遗留工程参考任务 (../server_old/)

- [ ] 阅读对标代码（仅参考，不修改）
- [ ] 提取设计思路和注意事项

### 3.3 任务依赖图

使用 Mermaid 格式绘制任务依赖图，例如：

```mermaid
graph TD
    A[协议工程: 新增消息] --> B[业务工程: 定义结构体]
    B --> C[业务工程: 实现 Service]
    C --> D[业务工程: 单元测试]
    D --> E[集成测试]

    F[配置工程: 新增表] --> G[配置生成]
    G --> H[业务工程: 加载配置]

    I[DB: Migration] --> J[本地测试 Migration]
```

### 3.4 输出任务清单

生成结构化的任务清单，包含：

```
## 任务清单

### 业务工程 (P1GoServer/)
- [ ] [TASK-001] 新增 XxManager 结构体定义
- [ ] [TASK-002] 实现 XxService.DoSomething()
- [ ] [TASK-003] 单元测试 TestXxService

### 协议工程 (../proto/)
- [ ] [TASK-101] 新增 XxRequest/XxResponse 消息
- [ ] [TASK-102] 运行 protoc 生成代码
- [ ] [TASK-103] 更新 git submodule

### 配置工程 (../config/)
- [ ] [TASK-201] 新增 XxConfig 配置表
- [ ] [TASK-202] 修改配置生成器

### 数据库
- [ ] [TASK-301] 编写 Migration 脚本
- [ ] [TASK-302] 本地测试 Migration

### 任务依赖
TASK-001 → TASK-002 → TASK-003 → 集成测试
TASK-101 → TASK-002
TASK-201 → TASK-202 → TASK-002
TASK-301 → 本地集成测试
```

保存到 `docs/tasks-[feature-name].md`，**等待用户确认后进入 Phase 4。**

---

## Phase 4：实现

### 4.1 实现策略

按照以下策略进行实现：

1. **协议优先**：先完成协议工程，生成协议代码（参考 `PROTO.md`）
2. **配置先行**：配置工程完成后，业务工程才能加载配置
3. **数据库就绪**：Migration 脚本在本地测试通过（参考 `DB.md`、`TEST.md`）
4. **业务最后**：在所有依赖就绪后实现业务逻辑

### 4.2 跨工程并行实现

不同工程的任务可以并行执行，启动多个 subagent：

#### 并行任务分组

**组1（可并行）：**
- 协议工程：新增消息定义
- 配置工程：新增配置表
- 数据库：编写 Migration 脚本

**组2（可并行，依赖组1）：**
- 协议工程：运行 protoc
- 配置工程：修改配置生成器
- 数据库：本地测试 Migration

**组3（串行，依赖组1和组2）：**
- 业务工程：基础设施（结构体定义）
- 业务工程：Service 层实现
- 业务工程：单元测试（参考 `TEST.md` 测试编写规范）
- 业务工程：集成测试（参考 `TEST.md` 测试编写规范）

### 4.3 代码实现规范

在实现过程中，严格遵守以下规范：

#### Go 编码规范

1. **错误处理**：
   - 不忽略错误（除非明确说明原因）
   - 使用 wrap 包装错误上下文
   - 提供有意义的错误信息

2. **并发安全**：
   - 共享变量必须加锁
   - 使用 defer 释放锁
   - 避免锁持有时间过长

3. **资源管理**：
   - 使用 defer 关闭资源（文件、连接等）
   - 避免资源泄漏
   - 合理使用 context 控制超时

#### 事务性代码规范

1. **先验证后执行**：在修改任何状态之前，先完成所有前置验证
   ```go
   // 正确：先验证
   if !canDeduct(player, cost) { return ErrInsufficientFunds }
   if !hasSpace(player.Bag) { return ErrBagFull }
   // 验证通过后再执行
   player.DeductMoney(cost)
   player.Bag.AddItem(item)
   ```

2. **保存快照用于回滚**：复杂操作前保存状态快照
   ```go
   snapshot := entity.Snapshot()
   defer func() {
       if err != nil { entity.Restore(snapshot) }
   }()
   ```

3. **使用 defer 确保清理**：锁释放、资源回收必须用 defer
   ```go
   lock.Lock()
   defer lock.Unlock()
   ```

4. **跨服务调用使用请求ID**：便于追踪和幂等性检查
   ```go
   reqID := uuid.New().String()
   resp, err := rpcClient.Call(ctx, reqID, req)
   ```

5. **记录关键操作日志**：便于问题排查和数据恢复
   ```go
   log.Info("transaction_start", "player_id", pid, "action", "trade", "req_id", reqID)
   // ... 执行操作
   log.Info("transaction_end", "player_id", pid, "action", "trade", "req_id", reqID, "result", "success")
   ```

**注意：**
- 每个 subagent 须阅读完整的设计方案以理解上下文
- 涉及相同文件的任务不要并行（避免冲突）
- 不同工程的任务可以并行（无文件冲突）
- 遇到阻塞问题暂停并汇报，不要自行绕过

全部实现完成后，**等待用户确认后进入 Phase 5。**

---

## Phase 5：构建与测试

### 5.1 构建验证

确保所有工程构建通过：

```bash
# 构建所有服务器
make build

# 代码检查
make lint
make fmt
```

### 5.2 测试执行

按照 `TEST.md` 规范执行测试：

1. **单元测试**：`make test`
2. **集成测试**：跨模块交互测试
3. **Migration 测试**：数据库变更脚本本地测试
4. **回归测试**：确保原有功能不受影响

### 5.3 验证清单

- [ ] `make build` 构建通过
- [ ] `make lint` 无错误
- [ ] `make test` 测试通过
- [ ] Migration 脚本本地测试通过
- [ ] 协议生成代码已更新
- [ ] 配置生成代码已更新

**全部验证通过后，等待用户确认后进入 Phase 6。**

---

## Phase 6：产出审查

> 详细审查规范、检查清单和审查报告模板参见 `REVIEW.md`

分两步进行：

### 第一步：多 Agent 并行审查代码

同时启动 4 个 subagent 审查实际代码产出（覆盖所有涉及的工程）：

1. **dev-workflow-code-reviewer**：审查代码质量、Rules 合规、编码规范（各工程）
2. **dev-workflow-security-reviewer**：审查安全性（注入、凭证泄露、越权访问、跨工程安全边界）
3. **dev-workflow-transaction-reviewer**：审查事务性实现（见下方检查清单）
4. **dev-workflow-test-designer**：评估测试覆盖是否充分（参考 `TEST.md` 中的评估要点和验证清单）

**事务性代码审查检查清单（dev-workflow-transaction-reviewer）**：

| 检查项 | 审查内容 |
|--------|----------|
| 验证顺序 | 是否先完成所有验证再执行状态变更？ |
| 回滚机制 | 失败路径是否正确回滚所有已变更的状态？ |
| 锁的使用 | 锁是否正确获取和释放？是否使用 defer？ |
| 并发安全 | 共享数据访问是否有竞态条件？ |
| 幂等实现 | 幂等键生成和检查逻辑是否正确？ |
| 超时处理 | 是否设置合理的超时？超时后是否正确处理？ |
| 日志记录 | 关键操作是否有足够的日志用于追踪？ |
| 错误处理 | 错误是否正确传播？是否有静默失败？ |
| 资源泄漏 | 是否有未释放的锁、未关闭的连接？ |

### 第二步：综合审查

对比设计方案和实际代码产出，进行全面核对：

1. 阅读设计方案，提取所有功能点和验收标准
2. 分工程运行 git diff 查看所有改动
3. 逐项核对：每个功能点是否已实现
4. 协议检查：协议定义是否与设计一致，生成文件是否最新
5. 配置检查：配置项是否与设计一致，各环境配置是否完整
6. DB 检查：表结构、索引、Migration 是否与设计一致
7. **事务性检查**：
   - 事务边界实现是否与设计一致？
   - 回滚逻辑是否完整覆盖所有失败路径？
   - 并发控制机制是否正确实现？
   - 幂等性检查是否到位？
8. 跨工程集成检查：工程间的数据流是否正确连通
9. 运行构建和测试：确认通过
10. 汇总所有 subagent 的审查结果

输出完整审查报告，包含：
- 功能完整性核对表
- 各工程改动汇总
- 协议一致性核对表
- 配置一致性核对表
- DB 一致性核对表
- **事务性一致性核对表**
- Agent 审查汇总
- 发现的问题和修复建议
- 测试覆盖情况（参考 `TEST.md` 验证清单）

**如有问题需修复：** 修复后再次运行审查，直到全部通过。

**等待用户确认后进入 Phase 7。**

---

## Phase 7：经验沉淀

回顾整个流程，将经验教训沉淀为可复用的知识：

### 7.1 沉淀目标分类

根据经验类型，沉淀到对应的文档：

| 经验类型 | 目标文档 | 示例 |
|----------|----------|------|
| 编码规范问题 | `.claude/rules/` | 命名规范、错误处理模式 |
| 数据库相关 | `DB.md` | 新的存储限制、查询优化技巧、Migration 经验 |
| 协议工程相关 | `PROTO.md` | 协议设计模式、新的客户端请求限制 |
| 测试相关 | `TEST.md` | 测试技巧、新的验证清单项 |
| 代码审查相关 | `REVIEW.md` | 新的检查清单项、审查模式 |
| 架构约定 | `CLAUDE.md` | 工程间协作规范、模块设计原则 |
| Agent 优化 | Agent Memory | 审查模式、常见问题识别 |

### 7.2 沉淀流程

1. **识别经验类型**：回顾本次开发中遇到的问题和解决方案
2. **匹配目标文档**：根据上表确定应沉淀到哪个文档
3. **更新文档**：使用 Edit 工具追加内容到对应文档
4. **验证完整性**：确保新增内容与现有内容一致

### 7.3 询问用户

询问用户：
- 本次流程中是否有需要记录的经验教训？
- 经验类型是什么？（数据库/协议/测试/审查/架构/编码规范）
- 是否需要新增或修改 Rules？
- 是否需要更新 Skill 辅助文档（DB.md、PROTO.md、TEST.md、REVIEW.md）？

根据用户反馈执行沉淀操作。

### 7.4 自动沉淀提醒

以下情况应主动提醒用户是否需要沉淀：

| 触发条件 | 建议沉淀到 |
|----------|-----------|
| 发现新的数据库存储限制 | `DB.md` Section 9 |
| 新增客户端请求接口 | `PROTO.md` 第二部分（客户端请求限制）|
| 新增测试技巧或工具 | `TEST.md` |
| 新增审查检查项 | `REVIEW.md` 对应 Section |
| 重复出现的编码问题 | `.claude/rules/` |
| 新的架构约定 | `CLAUDE.md` |

---

## 执行原则

- 每个 Phase 完成后暂停，输出摘要等待用户确认再继续
- 所有设计文档和审查记录写入业务工程的 docs/ 目录持久化
- **操作不同工程时必须明确标注目标路径**，不能混淆
- 遇到阻塞问题立即汇报，不自行绕过
- 每个 Phase 完成后简要输出当前进度

---

## 经验沉淀（历史案例）

> 本节是 Phase 7 沉淀的历史经验案例，供后续开发参考。新的经验教训应按照 Phase 7.1 的分类规则，沉淀到对应的辅助文档中。

### 协议工程修改规范

> **详细规范已整合到 `PROTO.md`**，包括：
> - 使用 Edit 工具而非 Write 工具
> - 协议生成脚本选择（Windows/Linux）
> - 协议/代码不同步的排查步骤
> - 子模块操作

---

### 复用现有系统的扩展模式

#### 场景隔离模式

当需要为新场景（如樱花校园）复用现有系统（如小镇时间系统）时，采用以下模式：

**1. 添加场景类型枚举**
```go
type TimeSceneType int
const (
    TimeSceneType_Town   TimeSceneType = 0
    TimeSceneType_Sakura TimeSceneType = 1
)
```

**2. 添加行为控制字段**
```go
type TimeMgr struct {
    // ... 原有字段
    enableSleep bool          // 控制是否启用特定行为
    sceneType   TimeSceneType // 控制协议填充
}
```

**3. 提供场景专用构造函数**
```go
func NewTimeMgr(scene common.Scene) *TimeMgr {
    return &TimeMgr{enableSleep: true, sceneType: TimeSceneType_Town}
}

func NewTimeMgrForSakura(scene common.Scene) *TimeMgr {
    return &TimeMgr{enableSleep: false, sceneType: TimeSceneType_Sakura}
}
```

**4. 在 scene_impl.go 的 switch 分支中初始化**
```go
switch sceneType := s.sceneType.(type) {
case *common.TownSceneInfo:
    timeMgr := time_mgr.NewTimeMgr(s)
    // ...
case *common.SakuraSceneInfo:
    timeMgr := time_mgr.NewTimeMgrForSakura(s)
    // ...
}
```

**5. 在行为代码中用条件分支隔离**
```go
if t.enableSleep {
    // 小镇特有逻辑
} else {
    // 樱花校园逻辑
}
```

**6. 协议同步时用 sceneType 填充对应字段**
```go
switch t.sceneType {
case TimeSceneType_Town:
    info.TownTimeInfo = t.townTime.ToProto()
case TimeSceneType_Sakura:
    info.SakuraTimeInfo = t.townTime.ToSakuraProto()
}
```

#### 审查清单

扩展现有系统时，需审查以下隔离点：

- [ ] 构造函数是否正确设置场景类型字段
- [ ] 初始化代码是否在正确的 switch 分支中
- [ ] 行为差异是否通过条件字段正确分支
- [ ] 协议同步是否填充到正确的字段
- [ ] 数据存储是否使用独立的方法和消息类型
- [ ] 原有场景的功能是否保持不变（回归测试）

