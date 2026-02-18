# GECS 核心概念指南

> **深入理解实体组件系统架构**

本指南解释了使 GECS 强大的基本概念。阅读后，你将了解如何使用 ECS 原则构建游戏并利用 GECS 的独特功能。

## 📋 前置条件

*   已完成[入门指南](GETTING_STARTED-zh-CN-translation.md)
*   基础 GDScript 知识
*   对 Godot 节点系统的理解

## 🎯 为什么使用 ECS？

### 传统面向对象编程的问题

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

### ECS 解决方案

ECS 将数据（组件）与逻辑（系统）分开，围绕三个核心概念提供清晰的组织：

1.  **实体** – 游戏对象的 ID 或"槽位"
2.  **组件** – 纯数据对象，用于定义状态（例如，速度、健康）
3.  **系统** – 处理具有特定组件的实体的逻辑

这种模式简化了组织、协作和重构。系统仅作用于相关的组件。实体可以自由地改变其构成，而不会破坏整体设计。

## 🏗️ GECS 架构

GECS 扩展了标准 ECS，增加了 Godot 特有的功能：

*   **与 Godot 节点的集成** \- 实体可以是场景，组件是资源
*   **世界管理** \- 实体和系统的中央协调
*   **ECS 单例** \- 查询和处理的全球访问点
*   **高级查询** \- 基于属性的过滤和关系支持
*   **关系系统** \- 定义实体之间的复杂关联

## 🎭 实体

### 实体基础

实体是你在 GECS 中处理的核心数据容器。它们是扩展自 `Entity.gd` 的 Godot 节点，用于存储组件和关系。

**在代码中创建实体：**

```gdscript
# Create entity class with components
class_name MyEntity extends Entity

func define_components() -> Array:
    return [C_Transform.new(), C_Velocity.new(Vector3.UP)]

# Use the entity
var e_my_entity = MyEntity.new()
ECS.world.add_entity(e_my_entity)
```

**实体预制体（推荐）：** 由于 GECS 与 Godot 集成，创建包含实体根节点的场景并保存为 `.tscn` 文件。这些"预制体"可以包含用于可视化的子节点，同时保持 ECS 数据组织结构。

```gdscript
# e_player.gd - Entity prefab
class_name Player
extends Entity

func on_ready():
    # Sync transform from scene to component
    var c_trs = get_component(C_Transform) as C_Transform
    if not c_trs:
        return
    transform_comp.transform = self.global_transform # This works because the TSCN base type is Node3D and we extend Node3D with Entity (Which itself extends from Node)
```

### 实体生命周期

实体具有受管理的生命周期：

1.  **初始化** \- 实体添加到世界，组件从 `component_resources 加载`
2.  **define\_components()** - 调用以通过代码添加组件
3.  **on\_ready()** - 设置初始状态，同步变换
4.  **on\_destroy()** - 移除前的清理工作
5.  **on\_disable()/on\_enable()** - 处理启用/禁用状态

> **注意：** 在 GECS v5.0 及以上版本中，实体逻辑应由系统处理，而非在实体方法中处理。实体是纯粹的数据容器。

### 实体命名规范

**GECS 在整个框架中遵循一致的命名模式：**

*   **类名** ：`ClassCase` 表示它们所代表的事物
*   **文件名** ：`e_entity_name.gd` 使用 snake\_case

**示例：**

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

### 实体作为粘合代码

实体可以作为初始化和连接点：

```gdscript
class_name Player
extends Entity

@onready var mesh_instance = $MeshInstance3D
@onready var collision_shape = $CollisionShape3D

func on_ready():
    # Connect scene nodes to components
    var c_sprite = get_component(C_Sprite)
    if c_sprite:
        sprite_comp.mesh_instance = mesh_instance

    # Sync editor-placed transform to component
    var c_trs = get_component(C_Transform)
    if c_trs:
        transform_comp.transform = self.global_transform
```

## 📦 组件

### 组件基础

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

### 组件设计原则

**仅数据：**

```gdscript
# ✅ Good - Pure data
class_name C_Health
extends Component

@export var current: float = 100.0
@export var maximum: float = 100.0
@export var regeneration_rate: float = 1.0
```

**无游戏逻辑：**

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

### 组件命名规范

**GECS 使用一致的 C\_前缀系统：**

*   **类名** ： `C_ComponentName` 在类命名格式中
*   **文件名** : `c_component_name.gd` 使用蛇形命名法
*   **组织方式** : 按功能在文件夹中分组

**示例:**

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

**文件组织:**

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

### 添加组件

**通过编辑器（推荐）：** 在 Inspector 中添加到实体的 `component_resources` 数组 - 当实体添加到世界时会自动加载。

**通过 define\_components():**

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

## ⚙️ 系统

### 系统基础

系统包含游戏逻辑，并根据组件查询处理实体。它们应该小而独立，专注于单一职责。

系统包含两个主要部分：

*   **查询** \- 根据组件/关系定义要处理的实体
*   **处理** \- 在实体上运行的函数

### 系统类型

**实体处理：**

```gdscript
class_name LifetimeSystem
extends System

func query() -> QueryBuilder:
    return q.with_all([C_Lifetime])

func process(entities: Array[Entity], components: Array, delta: float):
    for entity in entities:
        var c_lifetime = entity.get_component(C_Lifetime) as C_Lifetime
        c_lifetime.lifetime -= delta

        if c_lifetime.lifetime <= 0:
            cmd.remove_entity(entity)  # Queued via CommandBuffer, safe during iteration
```

**使用 iterate()优化批量处理**

```gdscript
class_name VelocitySystem
extends System

func query() -> QueryBuilder:
    # Use iterate() to get component arrays for faster access
    return q.with_all([C_Velocity]).iterate([C_Velocity])

func process(entities: Array[Entity], components: Array, delta: float):
    # components[0] contains all C_Velocity components
    var velocities = components[0]

    for i in entities.size():
        # Direct array access is faster than get_component()
        var position: Vector3 = entities[i].transform.origin
        position += velocities[i].velocity * delta
        entities[i].transform.origin = position
```

### 子系统

将相关的逻辑分组到一个系统文件中 - 所有子系统使用统一的签名：

```gdscript
class_name DamageSystem
extends System

func sub_systems():
    return [
        # [query, callable] - all use same unified process signature
        [
            q
            .with_all([C_Health, C_Damage]),
            damage_entities
        ],
        [
            q
            .with_all([C_Health])
            .with_none([C_Dead])
            .iterate([C_Health]),
            regenerate_health
        ]
    ]

func damage_entities(entities: Array[Entity], components: Array, delta: float):
    # Process entities with damage
    for entity in entities:
        var c_health = entity.get_component(C_Health)
        var c_damage = entity.get_component(C_Damage)
        c_health.current -= c_damage.amount
        entity.remove_component(c_damage)

        if c_health.current <= 0:
            entity.add_component(C_Dead.new())

func regenerate_health(entities: Array[Entity], components: Array, delta: float):
    # Batch process using component arrays from iterate()
    var healths = components[0]
    for i in entities.size():
        healths[i].current = min(healths[i].current + 1 * delta, healths[i].maximum)
```

### 系统依赖

通过依赖关系控制系统执行顺序：

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

### 系统命名规范

*   **类名** ：`SystemNameSystem`，使用类命名格式（TransformSystem，PhysicsSystem）
*   **文件名** ：`s_system_name.gd`（s\_transform.gd，s\_physics.gd）

### 系统生命周期

系统遵循 Godot 节点生命周期：

*   `setup()` - 系统添加到世界后的初始设置
*   `process(entities, components, delta)` - 每帧调用的统一方法用于匹配实体
*   系统组用于有序处理

## 🔍 查询系统

### 查询构建器

GECS 使用流畅的 API 来构建实体查询：

```gdscript
ECS.world.query
    .with_all([C_Health, C_Position])          # Must have all these components
    .with_any([C_Player, C_Enemy])             # Must have at least one of these
    .with_none([C_Dead, C_Disabled])           # Must not have any of these
    .with_relationship([r_attacking_player])    # Must have these relationships
    .without_relationship([r_fleeing])          # Must not have these relationships
    .with_reverse_relationship([r_parent_of])   # Must be target of these relationships
    .iterate([C_Health])                        # Fetch these components and add to components array for quick iteration
```

### 查询方法

**基本查询操作：**

```gdscript
var entities = query.execute()                    # Get matching entities
var filtered = query.matches(entity_list)         # Filter existing list
var combined = query.combine(another_query)       # Combine queries
```

### 查询类型说明

**with\_all** - 实体必须具有所有指定的组成部分：

```gdscript
# Find entities that can move and be damaged
q.with_all([C_Position, C_Velocity, C_Health])
```

**with\_any** - 实体必须至少包含一个组件：

```gdscript
# Find players or enemies (anything controllable)
q.with_any([C_Player, C_Enemy])
```

**with\_none** - 实体不能包含这些组件中的任何一个：

```gdscript
# Find living entities (exclude dead/disabled)
q.with_all([C_Health]).with_none([C_Dead, C_Disabled])
```

### 组件属性查询

基于组件数据值进行查询：

```gdscript
# Find entities with low health
q.with_all([{C_Health: {"current": {"_lt": 20}}}])

# Find fast-moving entities
q.with_all([{C_Velocity: {"speed": {"_gt": 100}}}])

# Find entities with specific states
q.with_all([{C_State: {"current_state": {"_eq": "attacking"}}}])
```

**支持的运算符：**

*   `_eq` - 等于
*   `_ne` - 不等于
*   `_gt` - 大于
*   `_lt` - 小于
*   `_gte` - 大于或等于
*   `_lte` - 小于或等于
*   `_in` - 值在列表中
*   `_nin` - 值不在列表中

## 🔗 关系

### 关系基础

关系将实体连接在一起以形成复杂关联。它们由以下组成：

*   **源** \- 具有关系的实体
*   **关系** \- 定义关系类型的组件
*   **目标** \- 被关联的实体或类型

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

### 关系查询

**特定关系：**

```gdscript
# Any entity that likes alice
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), e_alice)])

# Any entity that eats 5 apples
ECS.world.query.with_relationship([Relationship.new(C_Eats.new(5), e_apple)])

# Any entity that likes the Food type
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), Food)])
```

**通配符关系：**

```gdscript
# Any entity with any relation toward heather
ECS.world.query.with_relationship([Relationship.new(ECS.wildcard, e_heather)])

# Any entity that likes anything
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), ECS.wildcard)])

# Any entity with any relation to Enemy type
ECS.world.query.with_relationship([Relationship.new(ECS.wildcard, Enemy)])
```

**反向关系：**

```gdscript
# Find entities that are being liked by someone
ECS.world.query.with_reverse_relationship([Relationship.new(C_Likes.new(), ECS.wildcard)])
```

### 关系最佳实践

**重用关系对象：**

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

## 🌍 世界管理

### 世界生命周期

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

### 系统分组和处理顺序

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

**场景设置优势：**

*   **视觉组织** ：在 Godot 编辑器中查看系统层次结构
*   **轻松重新排序** : 在组之间拖动系统
*   **检查器配置** : 在编辑器中设置系统属性
*   **可重用场景** : 在项目之间共享系统配置

## 🔄 数据驱动架构

### 组合优于继承

通过组合简单组件而非复杂的继承来构建实体：

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

### 模块化系统设计

保持系统小巧专注：

```gdscript
# ✅ Focused systems
class_name MovementSystem extends System
# Only handles position updates

class_name CollisionSystem extends System
# Only handles collision detection

class_name HealthSystem extends System
# Only handles health changes
```

这确保了：

*   **更易于调试** \- 模块职责清晰分离
*   **更好的可复用性** \- 系统可跨不同实体类型工作
*   **简化的测试** \- 每个系统可独立测试
*   **性能优化** \- 系统可以单独进行分析和优化

## 🎯 下一步

现在你已经理解了 GECS 的核心概念：

1.  **将这些模式** 应用到你的项目中
2.  **尝试探索关系** 以处理复杂的实体交互
3.  **设计组件层次结构**以支持您的游戏需求
4.  **在[性能指南中学习优化技巧](PERFORMANCE_OPTIMIZATION-zh-CN-translation.md)**
5.  **在[最佳实践指南中掌握常见模式](BEST_PRACTICES-zh-CN-translation.md)**

## 📚 相关文档

*   **[入门指南](GETTING_STARTED-zh-CN-translation.md)** \- 创建你的第一个 ECS 项目
*   **[最佳实践](BEST_PRACTICES-zh-CN-translation.md)** \- 编写可维护的 ECS 代码
*   **[性能优化](PERFORMANCE_OPTIMIZATION-zh-CN-translation.md)** \- 让你的游戏运行更快速
*   **[故障排除](TROUBLESHOOTING-zh-CN-translation.md)** \- 解决常见问题

* * *

*理解 ECS 是从“事物是什么”转变为“事物有什么”以及“在它们上操作什么”。这种数据和逻辑的分离是可扩展游戏架构的关键。*