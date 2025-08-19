# GECS Core Concepts Guide
GECS 核心概念指南

> **Deep understanding of Entity Component System architecture
> 深入理解实体组件系统架构**

This guide explains the fundamental concepts that make GECS powerful. After reading this, you'll understand how to architect games using ECS principles and leverage GECS's unique features.
本指南解释了使 GECS 强大的基本概念。阅读后，你将了解如何使用 ECS 原则构建游戏并利用 GECS 的独特功能。

## 📋 Prerequisites
📋 前置条件

*   Completed [Getting Started Guide](GETTING_STARTED-zh-CN-dual.md)
    完成入门指南
*   Basic GDScript knowledge
    基础 GDScript 知识
*   Understanding of Godot's node system
    对 Godot 节点系统的理解

## 🎯 Why ECS?
🎯 为什么是 ECS？

### The Problem with Traditional OOP
传统面向对象编程的问题

Traditional object-oriented approaches often bundle data and behavior together. Over time, this can become unwieldy and force complicated inheritance structures:
传统面向对象方法通常将数据和行为捆绑在一起。随着时间的推移，这可能会变得难以管理，并迫使复杂的继承结构：

```gdscript
# ❌ Traditional OOP problems
class BaseCharacter:
    # Lots of shared code

class Player extends BaseCharacter:
    # Player-specific code mixed with shared code

class Enemy extends BaseCharacter:
    # Enemy-specific code, some overlap with Player

class Boss extends Enemy:
    # Even more inheritance complexity
```

### The ECS Solution
ECS 解决方案

ECS keeps data (components) separate from logic (systems), providing clear organization around three core concepts:
ECS 将数据（组件）与逻辑（系统）分开，围绕三个核心概念提供清晰的组织：

1.  **Entities** – IDs or "slots" for your game objects
    实体——游戏对象的 ID 或"槽位"
2.  **Components** – Pure data objects that define state (e.g., velocity, health)
    组件——纯粹的数据对象，用于定义状态（例如，速度、健康）
3.  **Systems** – Logic that processes entities with specific components
    系统——处理具有特定组件的实体的逻辑

This pattern simplifies organization, collaboration, and refactoring. Systems only act upon relevant components. Entities can freely change their makeup without breaking the overall design.
这种模式简化了组织、协作和重构。系统仅作用于相关的组件。实体可以自由地改变其构成，而不会破坏整体设计。

## 🏗️ GECS Architecture
🏗️ GECS 架构

GECS extends standard ECS with Godot-specific features:
GECS 扩展了标准 ECS，增加了 Godot 特有的功能：

*   **Integration with Godot nodes** - Entities can be scenes, Components are resources
    与 Godot 节点的集成 - 实体可以是场景，组件是资源
*   **World management** - Central coordination of entities and systems
    世界管理 - 实体和系统的中央协调
*   **ECS singleton** - Global access point for queries and processing
    ECS 单例 - 查询和处理的全球访问点
*   **Advanced queries** - Property-based filtering and relationship support
    高级查询 - 基于属性的过滤和关系支持
*   **Relationship system** - Define complex associations between entities
    关系系统 - 定义实体之间的复杂关联

## 🎭 Entities
🎭 实体

### Entity Fundamentals
实体基础

Entities are the core data containers you work with in GECS. They're Godot nodes extending `Entity.gd` that hold components and relationships.
实体是你在 GECS 中处理的核心数据容器。它们是扩展自 `Entity.gd` 的 Godot 节点，用于存储组件和关系。

**Creating Entities in Code:
在代码中创建实体：**

```gdscript
# Create entity class with components
class_name MyEntity extends Entity

func define_components() -> Array:
    return [C_Transform.new(), C_Velocity.new(Vector3.UP)]

# Use the entity
var e_my_entity = MyEntity.new()
ECS.world.add_entity(e_my_entity)
```

**Entity Prefabs (Recommended):** Since GECS integrates with Godot, create scenes with Entity root nodes and save as `.tscn` files. These "prefabs" can include child nodes for visualization while maintaining ECS data organization.
实体预制体（推荐）：由于 GECS 与 Godot 集成，创建包含实体根节点的场景并保存为 `.tscn` 文件。这些"预制体"可以包含用于可视化的子节点，同时保持 ECS 数据组织。

```gdscript
# e_player.gd - Entity prefab
class_name Player
extends Entity

func on_ready():
    # Sync transform from scene to component
    if has_component(C_Transform):
        var transform_comp = get_component(C_Transform)
        transform_comp.transform = global_transform
```

### Entity Lifecycle
实体生命周期

Entities have a managed lifecycle:
实体具有受管理的生命周期：

1.  **Initialization** - Entity added to world, components loaded from `component_resources`
    初始化 - 实体添加到世界，组件从 `component_resources` 加载
2.  **define\_components()** - Called to add components via code
    define\_components() - 调用以通过代码添加组件
3.  **on\_ready()** - Setup initial states, sync transforms
    on\_ready() - 设置初始状态，同步变换
4.  **on\_update(delta)** - Called by systems each frame
    on\_update(delta) - 每帧由系统调用
5.  **on\_destroy()** - Cleanup before removal
    on\_destroy() - 移除前的清理
6.  **on\_disable()/on\_enable()** - Handle enable/disable states
    on\_disable()/on\_enable() - 处理启用/禁用状态

### Entity Naming Conventions
实体命名规范

**GECS follows consistent naming patterns throughout the framework:
GECS 在整个框架中遵循一致的命名模式：**

*   **Class names**: `ClassCase` representing the thing they are
    类名： `ClassCase` 代表它们所表示的事物
*   **File names**: `e_entity_name.gd` using snake\_case
    文件名： `e_entity_name.gd` 使用 snake\_case

**Examples:
示例：**

```gdscript
# e_player.gd
class_name Player extends Entity

# e_enemy.gd
class_name Enemy extends Entity

# e_projectile.gd
class_name Projectile extends Entity

# e_pickup_item.gd
class_name PickupItem extends Entity
```

### Entity as Glue Code
实体作为粘合代码

Entities can serve as initialization and connection points:
实体可以作为初始化和连接点：

```gdscript
class_name Player
extends Entity

@onready var mesh_instance = $MeshInstance3D
@onready var collision_shape = $CollisionShape3D

func on_ready():
    # Connect scene nodes to components
    var sprite_comp = get_component(C_Sprite)
    if sprite_comp:
        sprite_comp.mesh_instance = mesh_instance

    # Sync editor-placed transform to component
    if has_component(C_Transform):
        var transform_comp = get_component(C_Transform)
        transform_comp.transform = global_transform
```

## 📦 Components
📦 组件

### Component Fundamentals
组件基础

Components are pure data containers - they store state but contain no game logic. They can emit signals for reactive systems.
组件是纯粹的数据容器——它们存储状态但不包含游戏逻辑。它们可以为响应式系统发出信号。

```gdscript
# c_health.gd - Example component
class_name C_Health
extends Component

signal health_changed

## How much total health this entity has
@export var maximum := 100.0
## The current health value
@export var current := 100.0

func _init(max_health: float = 100.0):
    maximum = max_health
    current = max_health
```

### Component Design Principles
组件设计原则

**Data Only:
仅数据：**

```gdscript
# ✅ Good - Pure data
class_name C_Health
extends Component

@export var current: float = 100.0
@export var maximum: float = 100.0
@export var regeneration_rate: float = 1.0
```

**No Game Logic:
无游戏逻辑：**

```gdscript
# ❌ Avoid - Logic in components
class_name C_Health
extends Component

@export var current: float = 100.0

func take_damage(amount: float):  # This belongs in a system!
    current -= amount
    if current <= 0:
        print("Entity died!")
```

### Component Naming Conventions
组件命名规范

**GECS uses a consistent C\_ prefix system:
GECS 使用一致的 C\_前缀系统：**

*   **Class names**: `C_ComponentName` in ClassCase
    类名：使用 ClassCase 格式
*   **File names**: `c_component_name.gd` in snake\_case
    文件名：使用 snake\_case 格式
*   **Organization**: Group by purpose in folders
    组织：按用途在文件夹中分组

**Examples:
示例：**

```gdscript
# c_health.gd
class_name C_Health extends Component

# c_transform.gd
class_name C_Transform extends Component

# c_velocity.gd
class_name C_Velocity extends Component

# c_user_input.gd
class_name C_UserInput extends Component

# c_sprite_renderer.gd
class_name C_SpriteRenderer extends Component
```

**File Organization:
文件组织：**

```
components/
├── gameplay/
│   ├── c_health.gd
│   ├── c_damage.gd
│   └── c_inventory.gd
├── physics/
│   ├── c_transform.gd
│   ├── c_velocity.gd
│   └── c_collision.gd
└── rendering/
    ├── c_sprite.gd
    └── c_mesh.gd
```

### Adding Components
添加组件

**Via Editor (Recommended):** Add to entity's `component_resources` array in Inspector - these auto-load when entity is added to world.
通过编辑器（推荐）：在检查器中添加到实体的 `component_resources` 数组中——这些会在实体添加到世界时自动加载。

**Via define\_components():
通过 define\_components()：**

```gdscript
# e_player.gd - Define components programmatically
class_name Player
extends Entity

func define_components() -> Array:
    return [
        C_Health.new(100),
        C_Transform.new(),
        C_Input.new()
    ]

# Via Inspector: Add to component_resources array
# Components automatically loaded when entity added to world

# Dynamic addition (less common):
var entity = Player.new()
entity.add_component(C_StatusEffect.new("poison"))
ECS.world.add_entity(entity)
```

## ⚙️ Systems
⚙️ 系统

### System Fundamentals
系统基础

Systems contain game logic and process entities based on component queries. They should be small, atomic, and focused on one responsibility.
系统包含游戏逻辑，并根据组件查询处理实体。它们应该是小型的、原子的，并且专注于单一职责。

Systems have two main parts:
系统有两个主要部分：

*   **Query** - Defines which entities to process based on components/relationships
    查询 - 根据组件/关系定义要处理的实体
*   **Process** - The function that runs on entities
    处理 - 在实体上运行的函数

### System Types
系统类型

**Single Entity Processing:
单实体处理：**

```gdscript
class_name LifetimeSystem
extends System

func query() -> QueryBuilder:
    return q.with_all([C_Lifetime])

func process(entity: Entity, delta: float):
    var c_lifetime = entity.get_component(C_Lifetime) as C_Lifetime
    c_lifetime.lifetime -= delta

    if c_lifetime.lifetime <= 0:
        ECS.world.remove_entity(entity)
```

**Batch Processing (More Efficient):
批量处理（更高效）：**

```gdscript
class_name TransformSystem
extends System

func query() -> QueryBuilder:
    return q.with_all([C_Transform])

func process_all(entities: Array, _delta):
    var transforms = ECS.get_components(entities, C_Transform)
    for i in range(entities.size()):
        entities[i].global_transform = transforms[i].transform
```

### Sub-Systems
子系统

Group related logic into one system file:
将相关逻辑整合到一个系统文件中：

```gdscript
class_name DamageSystem
extends System

func sub_systems():
    return [
        # [query, callable, process_all_flag]
        [ECS.world.query.with_all([C_Health, C_Damage]), damage_entities, false],
        [ECS.world.query.with_all([C_Health]).with_none([C_Dead]), regenerate_health, true]
    ]

func damage_entities(entity: Entity, delta: float):
    var health = entity.get_component(C_Health)
    var damage = entity.get_component(C_Damage)
    health.current -= damage.amount
    entity.remove_component(damage)
    
    if health.current <= 0:
        entity.add_component(C_Dead.new())

func regenerate_health(entities: Array, delta: float):
    # Batch process health regeneration
    var healths = ECS.get_components(entities, C_Health)
    for health in healths:
        health.current = min(health.current + 1 * delta, health.maximum)
```

### System Dependencies
系统依赖

Control system execution order with dependencies:
通过依赖控制系统执行顺序：

```gdscript
class_name RenderSystem
extends System

func deps() -> Dictionary[int, Array]:
    return {
        Runs.After: [MovementSystem, TransformSystem],  # Run after these
        Runs.Before: [UISystem]  # Run before this
    }

# Special case: run after ALL other systems
class_name TransformSystem
extends System

func deps() -> Dictionary[int, Array]:
    return {
        Runs.After: [ECS.wildcard]  # Runs after everything else
    }
```

### System Naming Conventions
系统命名规范

*   **Class names**: `SystemNameSystem` in ClassCase (TransformSystem, PhysicsSystem)
    类名： `SystemNameSystem` 在 ClassCase (TransformSystem, PhysicsSystem) 中
*   **File names**: `s_system_name.gd` (s\_transform.gd, s\_physics.gd)
    文件名： `s_system_name.gd` (s\_transform.gd, s\_physics.gd)

### System Lifecycle
系统生命周期

Systems follow Godot node lifecycle:
系统遵循 Godot 节点生命周期：

*   `on_ready()` - Initial setup after components loaded
    `on_ready()` - 组件加载后的初始设置
*   `process(delta)` or `process_all(entities, delta)` - Called each frame for matching entities
    `process(delta)` 或 `process_all(entities, delta)` - 每帧调用以匹配实体
*   System groups for organized processing order
    系统组用于有序处理

## 🔍 Query System
🔍 查询系统

### Query Builder
查询构建器

GECS uses a fluent API for building entity queries:
GECS 使用一种流畅的 API 来构建实体查询：

```gdscript
ECS.world.query
    .with_all([C_Health, C_Position])          # Must have all these components
    .with_any([C_Player, C_Enemy])             # Must have at least one of these
    .with_none([C_Dead, C_Disabled])           # Must not have any of these
    .with_relationship([r_attacking_player])    # Must have these relationships
    .without_relationship([r_fleeing])          # Must not have these relationships
    .with_reverse_relationship([r_parent_of])   # Must be target of these relationships
```

### Query Methods
查询方法

**Basic Query Operations:
基本查询操作：**

```gdscript
var entities = query.execute()                    # Get matching entities
var filtered = query.matches(entity_list)         # Filter existing list
var combined = query.combine(another_query)       # Combine queries
```

### Query Types Explained
查询类型说明

**with\_all** - Entities must have ALL specified components:
with\_all - 实体必须包含所有指定组件：

```gdscript
# Find entities that can move and be damaged
q.with_all([C_Position, C_Velocity, C_Health])
```

**with\_any** - Entities must have AT LEAST ONE of the components:
with\_any - 实体必须至少包含一个组件：

```gdscript
# Find players or enemies (anything controllable)
q.with_any([C_Player, C_Enemy])
```

**with\_none** - Entities must NOT have any of these components:
with\_none - 实体不能包含这些组件：

```gdscript
# Find living entities (exclude dead/disabled)
q.with_all([C_Health]).with_none([C_Dead, C_Disabled])
```

### Component Property Queries
组件属性查询

Query based on component data values:
基于组件数据值的查询：

```gdscript
# Find entities with low health
q.with_all([{C_Health: {"current": {"_lt": 20}}}])

# Find fast-moving entities
q.with_all([{C_Velocity: {"speed": {"_gt": 100}}}])

# Find entities with specific states
q.with_all([{C_State: {"current_state": {"_eq": "attacking"}}}])
```

**Supported Operators:
支持的运算符：**

*   `_eq` - Equal to
    `_eq` - 等于
*   `_ne` - Not equal to
    `_ne` - 不等于
*   `_gt` - Greater than
    `_gt` - 大于
*   `_lt` - Less than
    `_lt` - 小于
*   `_gte` - Greater than or equal
    `_gte` - 大于或等于
*   `_lte` - Less than or equal
    `_lte` - 小于或等于
*   `_in` - Value in list
    `_in` - 列表中的值
*   `_nin` - Value not in list
    `_nin` - 列表中没有的值

## 🔗 Relationships
🔗 关系

### Relationship Fundamentals
关系基础

Relationships link entities together for complex associations. They consist of:
关系将实体连接起来形成复杂的关联。它们由以下组成：

*   **Source** - Entity that has the relationship
    源 - 具有关系的实体
*   **Relation** - Component defining the relationship type
    关系 - 定义关系类型的组件
*   **Target** - Entity or type being related to
    目标 - 被关联的实体或类型

```gdscript
# Create relationship components
class_name C_Likes extends Component
class_name C_Loves extends Component
class_name C_Eats extends Component
@export var quantity: int = 1

# Create entities
var e_bob = Entity.new()
var e_alice = Entity.new()
var e_heather = Entity.new()
var e_apple = Food.new()

# Add relationships
e_bob.add_relationship(Relationship.new(C_Likes.new(), e_alice))        # bob likes alice
e_alice.add_relationship(Relationship.new(C_Loves.new(), e_heather))    # alice loves heather
e_heather.add_relationship(Relationship.new(C_Likes.new(), Food))       # heather likes food (type)
e_heather.add_relationship(Relationship.new(C_Eats.new(5), e_apple))    # heather eats 5 apples
```

### Relationship Queries
关系查询

**Specific Relationships:
特定关系：**

```gdscript
# Any entity that likes alice
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), e_alice)])

# Any entity that eats 5 apples
ECS.world.query.with_relationship([Relationship.new(C_Eats.new(5), e_apple)])

# Any entity that likes the Food type
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), Food)])
```

**Wildcard Relationships:
通配符关系：**

```gdscript
# Any entity with any relation toward heather
ECS.world.query.with_relationship([Relationship.new(ECS.wildcard, e_heather)])

# Any entity that likes anything
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), ECS.wildcard)])

# Any entity with any relation to Enemy type
ECS.world.query.with_relationship([Relationship.new(ECS.wildcard, Enemy)])
```

**Reverse Relationships:
反向关系：**

```gdscript
# Find entities that are being liked by someone
ECS.world.query.with_reverse_relationship([Relationship.new(C_Likes.new(), ECS.wildcard)])
```

### Relationship Best Practices
关系最佳实践

**Reuse Relationship Objects:
重用关系对象：**

```gdscript
# Reuse for performance
var r_likes_apples = Relationship.new(C_Likes.new(), e_apple)
var r_attacking_players = Relationship.new(C_IsAttacking.new(), Player)

# Consider a static relationships class
class_name Relationships

static func attacking_players():
    return Relationship.new(C_IsAttacking.new(), Player)

static func chasing_anything():
    return Relationship.new(C_IsChasing.new(), ECS.wildcard)
```

## 🌍 World Management
🌍 世界管理

### World Lifecycle
世界生命周期

The World is the central manager for all entities and systems:
世界是所有实体和系统的中央管理者：

```gdscript
# main.gd - Simple scene-based setup
extends Node

@onready var world: World = $World

func _ready():
    Bootstrap.bootstrap()  # Initialize game-specific setup
    ECS.world = world
    # Systems are automatically registered via scene composition

# Process systems by groups in order
func _process(delta):
    world.process(delta, "run-first")  # Initialization
    world.process(delta, "input")      # Input handling
    world.process(delta, "gameplay")   # Game logic
    world.process(delta, "ui")         # UI updates
    world.process(delta, "run-last")   # Cleanup

func _physics_process(delta):
    world.process(delta, "physics")    # Physics systems
    world.process(delta, "debug")      # Debug systems
```

### System Groups and Processing Order
系统组和处理顺序

Organize systems using scene-based composition with execution groups:
使用基于场景的组合和执行组来组织系统：

```
default_systems.tscn Structure:
├── run-first (SystemGroup)
│   ├── VictimInitSystem
│   └── EcsStorageLoad
├── input (SystemGroup)
│   ├── ItemSystem
│   ├── WeaponsSystem
│   └── PlayerControlsSystem
├── gameplay (SystemGroup)
│   ├── GearSystem
│   ├── DeathSystem
│   └── EventSystem
├── physics (SystemGroup)
│   ├── FrictionSystem
│   ├── CharacterBody3DSystem
│   └── TransformSystem
├── ui (SystemGroup)
│   └── UiVisibilitySystem
├── debug (SystemGroup)
│   └── DebugLabel3DSystem
└── run-last (SystemGroup)
    ├── ActionsSystem
    └── PendingDeleteSystem
```

**Scene Setup Benefits:
场景设置优势：**

*   **Visual Organization**: See system hierarchy in Godot editor
    视觉组织：在 Godot 编辑器中查看系统层级
*   **Easy Reordering**: Drag systems between groups
    轻松重新排序：拖动系统在不同组之间
*   **Inspector Configuration**: Set system properties in editor
    检查器配置：在编辑器中设置系统属性
*   **Reusable Scenes**: Share system configurations between projects
    可复用场景：在项目间共享系统配置

## 🔄 Data-Driven Architecture
🔄 数据驱动架构

### Composition Over Inheritance
组合优于继承

Build entities by combining simple components rather than complex inheritance:
通过组合简单组件来构建实体，而不是复杂的继承：

```gdscript
# ✅ Composition approach in entity definition
class_name Player extends Entity

func define_components() -> Array:
    return [
        C_Health.new(100),
        C_Movement.new(200.0),
        C_Input.new(),
        C_Inventory.new()
    ]

# Same components reused for different entity types
enemy.add_component(C_Health.new(50))
enemy.add_component(C_Movement.new(100.0))
enemy.add_component(C_AI.new())
enemy.add_component(C_Sprite.new("enemy.png"))
```

### Modular System Design
模块化系统设计

Keep systems small and focused:
保持系统小而专注：

```gdscript
# ✅ Focused systems
class_name MovementSystem extends System
# Only handles position updates

class_name CollisionSystem extends System
# Only handles collision detection

class_name HealthSystem extends System
# Only handles health changes
```

This ensures:
这确保了：

*   **Easier debugging** - Clear separation of concerns
    更易于调试 - 聚焦于明确的责任划分
*   **Better reusability** - Systems work across different entity types
    更好的可重用性 - 系统可跨不同实体类型工作
*   **Simplified testing** - Each system can be tested independently
    简化测试 - 每个系统可独立测试
*   **Performance optimization** - Systems can be profiled and optimized individually
    性能优化 - 系统可单独进行性能分析和优化

## 🎯 Next Steps
🎯 下一步

Now that you understand GECS's core concepts:
现在你已经理解了 GECS 的核心概念：

1.  **Apply these patterns** in your own projects
    将这些模式应用于你自己的项目
2.  **Experiment with relationships** for complex entity interactions
    尝试探索复杂实体交互的关系
3.  **Design component hierarchies** that support your game's needs
    设计支持你游戏需求的组件层次结构
4.  **Learn optimization techniques** in [Performance Guide](PERFORMANCE_OPTIMIZATION-zh-CN-dual.md)
    学习性能指南中的优化技巧
5.  **Master common patterns** in [Best Practices Guide](BEST_PRACTICES-zh-CN-dual.md)
    掌握最佳实践指南中的常见模式

## 📚 Related Documentation
📚 相关文档

*   **[Getting Started](GETTING_STARTED-zh-CN-dual.md)** - Build your first ECS project
    入门指南 - 构建你的第一个 ECS 项目
*   **[Best Practices](BEST_PRACTICES-zh-CN-dual.md)** - Write maintainable ECS code
    最佳实践 - 编写可维护的 ECS 代码
*   **[Performance Optimization](PERFORMANCE_OPTIMIZATION-zh-CN-dual.md)** - Make your games run fast
    性能优化 - 让你的游戏运行快速
*   **[Troubleshooting](TROUBLESHOOTING-zh-CN-dual.md)** - Solve common issues
    故障排除 - 解决常见问题

* * *

*"Understanding ECS is about shifting from 'what things are' to 'what things have' and 'what operates on them.' This separation of data and logic is the key to scalable game architecture."
"理解 ECS 的关键在于从'事物是什么'转变为'事物拥有什么'以及'对它们进行操作什么'。这种数据和逻辑的分离是可扩展游戏架构的关键。"*
