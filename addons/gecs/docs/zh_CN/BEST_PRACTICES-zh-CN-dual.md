# GECS Best Practices Guide
GECS 最佳实践指南

> **Write maintainable, performant ECS code
> 编写可维护、高性能的 ECS 代码**

This guide covers proven patterns and practices for building robust games with GECS. Apply these patterns to keep your code clean, fast, and easy to debug.
本指南涵盖了使用 GECS 构建稳健游戏的最佳模式和实践。应用这些模式，可以使您的代码保持整洁、快速且易于调试。

## 📋 Prerequisites
📋 先决条件

*   Completed [Getting Started Guide](GETTING_STARTED.md)
    完成入门指南
*   Understanding of [Core Concepts](CORE_CONCEPTS.md)
    理解核心概念

## 🧱 Component Design Patterns
🧱 组件设计模式

### Keep Components Pure Data
保持组件的数据纯净

Components should only hold data, never logic or behavior.
组件应仅持有数据，从不包含逻辑或行为。

```gdscript
# ✅ Good - Pure data component
class_name C_Health
extends Component

@export var current: float = 100.0
@export var maximum: float = 100.0
@export var regeneration_rate: float = 1.0

func _init(max_health: float = 100.0):
    maximum = max_health
    current = max_health
```

```gdscript
# ❌ Avoid - Logic in components
class_name C_Health
extends Component

@export var current: float = 100.0
@export var maximum: float = 100.0

# This belongs in a system, not a component
func take_damage(amount: float):
    current -= amount
    if current <= 0:
        print("Entity died!")
```

### Use Composition Over Inheritance
优先使用组合而非继承

Build entities by combining simple components rather than complex inheritance hierarchies.
通过组合简单的组件来构建实体，而不是使用复杂的继承层次结构。

```gdscript
# ✅ Good - Composable components via define_components() or scene setup
class_name Player
extends Entity

func define_components() -> Array:
    return [
        C_Health.new(100),
        C_Transform.new(),
        C_Input.new()
    ]

class_name Enemy
extends Entity

func define_components() -> Array:
    return [
        C_Health.new(50),
        C_Transform.new(),
        C_AI.new()
    ]
```

### Design for Configuration
设计便于配置

Make components easily configurable through export properties.
通过导出属性轻松配置组件。

```gdscript
# ✅ Good - Configurable component
class_name C_Movement
extends Component

@export var speed: float = 100.0
@export var acceleration: float = 500.0
@export var friction: float = 800.0
@export var max_speed: float = 300.0
@export var can_fly: bool = false

func _init(spd: float = 100.0, can_fly_: bool = false):
    speed = spd
    can_fly = can_fly_
```

## ⚙️ System Design Patterns
⚙️ 系统设计模式

### Single Responsibility Principle
单一职责原则

Each system should handle one specific concern.
每个系统应处理一个特定的问题。

```gdscript
# ✅ Good - Focused systems
class_name MovementSystem extends System
func query(): return q.with_all([C_Position, C_Velocity])

class_name RenderSystem extends System
func query(): return q.with_all([C_Position, C_Sprite])

class_name HealthSystem extends System
func query(): return q.with_all([C_Health])
```

### Use System Groups for Processing Order
使用系统组处理订单

Organize systems into logical groups using scene-based organization. Systems are grouped in scene nodes and processed in the correct order.
根据场景组织将系统组织成逻辑组。系统会在场景节点中分组并按正确的顺序处理。

```gdscript
# main.gd - Process systems in correct order
func _process(delta):
    world.process(delta, "run-first")  # Initialization systems
    world.process(delta, "input")      # Input handling
    world.process(delta, "gameplay")   # Game logic
    world.process(delta, "ui")         # UI updates
    world.process(delta, "run-last")   # Cleanup systems

func _physics_process(delta):
    world.process(delta, "physics")    # Physics systems
    world.process(delta, "debug")      # Debug systems
```

### Early Exit for Performance
性能提前退出

Return early from system processing when no work is needed.
在系统处理中无需工作时提前返回。

```gdscript
# ✅ Good - Early exit patterns
class_name HealthRegenerationSystem extends System

func query():
    return q.with_all([C_Health]).with_none([C_Dead])

func process(entity: Entity, delta: float):
    var health = entity.get_component(C_Health)

    # Early exit if already at max health
    if health.current >= health.maximum:
        return

    # Apply regeneration
    health.current = min(health.current + health.regeneration_rate * delta, health.maximum)
```

## 🏗️ Code Organization Patterns
代码组织模式

### GECS Naming Conventions
GECS 命名规范

```gdscript
# ✅ GECS Standard naming patterns:

# Components: C_ComponentName class, c_component_name.gd file
class_name C_Health extends Component      # c_health.gd
class_name C_Position extends Component    # c_position.gd

# Systems: SystemNameSystem class, s_system_name.gd file
class_name MovementSystem extends System   # s_movement.gd
class_name RenderSystem extends System     # s_render.gd

# Entities: EntityName class, e_entity_name.gd file
class_name Player extends Entity           # e_player.gd
class_name Enemy extends Entity            # e_enemy.gd

# Observers: ObserverNameObserver class, o_observer_name.gd file
class_name HealthUIObserver extends Observer  # o_health_ui.gd
```

### File Organization
文件组织

Organize your ECS files by theme for better scalability:
按主题组织你的 ECS 文件以提高可扩展性：

```
project/
├── components/
│   ├── ai/              # AI-related components
│   ├── animation/       # Animation components
│   ├── gameplay/        # Core gameplay components
│   ├── gear/           # Equipment/gear components
│   ├── item/           # Item system components
│   ├── multiplayer/    # Multiplayer-specific
│   ├── relationships/  # Relationship components
│   ├── rendering/      # Visual/rendering
│   └── weapon/         # Weapon system
├── entities/
│   ├── enemies/        # Enemy entities
│   ├── gameplay/       # Core entities
│   ├── items/          # Item entities
│   └── ui/             # UI entities
├── systems/
│   ├── combat/         # Combat systems
│   ├── core/           # Core ECS systems
│   ├── gameplay/       # Gameplay systems
│   ├── input/          # Input systems
│   ├── interaction/    # Interaction systems
│   ├── physics/        # Physics systems
│   └── ui/             # UI systems
└── observers/
    └── o_transform.gd   # Reactive systems
```

## 🎮 Common Game Patterns
🎮 常见游戏模式

### Player Character Pattern
玩家角色模式

```gdscript
# e_player.gd
class_name Player
extends Entity

func on_ready():
    # Common pattern: sync scene transform to component
    if has_component(C_Transform):
        var transform_comp = get_component(C_Transform)
        transform_comp.transform = global_transform
    add_to_group("player")
```

### Enemy Pattern
敌人模式

```gdscript
# e_enemy.gd
class_name Enemy
extends Entity

func on_ready():
    # Sync transform and add to enemy group
    if has_component(C_Transform):
        var transform_comp = get_component(C_Transform)
        transform_comp.transform = global_transform
    add_to_group("enemies")
```

## 🚀 Performance Best Practices
🚀 性能最佳实践

### Use Batch Processing for Performance
使用批量处理提高性能

```gdscript
# ✅ Good - Batch processing with process_all
class_name TransformSystem
extends System

func query():
    return q.with_all([C_Transform])

func process_all(entities: Array, _delta):
    # Batch access to components for better performance
    var transforms = ECS.get_components(entities, C_Transform)
    for i in range(entities.size()):
        entities[i].global_transform = transforms[i].transform
```

### Use Specific Queries
使用特定查询

```gdscript
# ✅ Good - Specific query
class_name PlayerInputSystem extends System
func query():
    return q.with_all([C_Input, C_Movement]).with_group("player")

# ❌ Avoid - Overly broad queries
class_name UniversalMovementSystem extends System
func query():
    return q.with_all([C_Transform])  # Too broad - matches everything
```

## 🎭 Entity Prefabs (Scene Files)
🎭 实体预制件（场景文件）

### Using Godot Scenes as Entity Prefabs
使用 Godot 场景作为实体预制件

The most powerful pattern in GECS is using Godot's scene system (.tscn files) as entity prefabs. This combines ECS data with Godot's visual editor:
GECS 中最强大的模式是使用 Godot 的场景系统（.tscn 文件）作为实体预制件。这将 ECS 数据与 Godot 的视觉编辑器结合在一起：

```
e_player.tscn Structure:
├── Player (Entity node - extends your e_player.gd class)
│   ├── MeshInstance3D (visual representation)
│   ├── CollisionShape3D (physics collision)
│   ├── AudioStreamPlayer3D (sound effects)
│   └── SkeletonAttachment3D (for equipment)
```

**Benefits of Scene-based Prefabs:
基于场景的预制件的优势：**

*   **Visual Editing**: Design entities in Godot's 3D editor
    视觉编辑：在 Godot 的 3D 编辑器中设计实体
*   **Component Assignment**: Set up ECS components in the Inspector
    组件分配：在检查器中设置 ECS 组件
*   **Godot Integration**: Leverage existing Godot nodes and systems
    Godot 集成：利用现有的 Godot 节点和系统
*   **Reusability**: Instantiate the same prefab multiple times
    可重用性：多次实例化同一个预制件
*   **Version Control**: Scene files work well with git
    版本控制：场景文件与 git 配合得很好

**Setting up Entity Prefabs:
设置实体预制件：**

1.  **Create scene with Entity as root**: `e_player.tscn` with `Player` entity node.
    以 Entity 作为根节点创建场景： `e_player.tscn` ，包含 `Player` 实体节点。
    *   Another trick here is to add a CharacterBody3d and then extend that CharacterBody3D with the e\_player.gd script this way you get Entity class and CharacterBody3D class data
        这里还有一个技巧是添加一个 CharacterBody3d，然后通过 e\_player.gd 脚本扩展 CharacterBody3D，这样你就可以获得 Entity 类和 CharacterBody3D 类的数据。
2.  **Add visual/physics children**: Add MeshInstance3D, CollisionShape3D, etc. as children
    添加视觉/物理子节点：将 MeshInstance3D、CollisionShape3D 等作为子节点添加。
3.  **Configure components in Inspector**: Add components to the `component_resources` array
    在检查器中配置组件：将组件添加到 `component_resources` 数组中
4.  **Save as reusable prefab**: Save the .tscn file for instantiation
    保存为可重用预制体：保存 .tscn 文件以便实例化
5.  **Set up on\_ready()**: Handle any initialization logic
    设置 on\_ready()：处理任何初始化逻辑

### Component Assignment in Prefabs
预制体中的组件分配

**Method 1: Inspector Assignment (Recommended)
方法 1：Inspector 分配（推荐）**

Set up components directly in the Godot Inspector:
直接在 Godot Inspector 中设置组件：

```gdscript
# In e_player.tscn entity root node Inspector:
# Component Resources array:
# - [0] C_Health.new() (max: 100, current: 100)
# - [1] C_Transform.new() (synced with scene transform)
# - [2] C_Input.new() (for player controls)
# - [3] C_LocalPlayer.new() (mark as local player)
```

**Method 2: define\_components() (Programmatic)
方法 2：define\_components()（程序化）**

```gdscript
# e_player.gd attached to Player.tscn root
class_name Player
extends Entity

func define_components() -> Array:
    return [
        C_Health.new(100),
        C_Transform.new(),
        C_Input.new(),
        C_LocalPlayer.new()
    ]

func on_ready():
    # Initialize after components are ready
    if has_component(C_Transform):
        var transform_comp = get_component(C_Transform)
        transform_comp.transform = global_transform
    add_to_group("player")
```

**Method 3: Hybrid Approach
方法 3：混合方法**

```gdscript
# Core components via Inspector, dynamic components via script
func on_ready():
    # Sync scene transform to component
    if has_component(C_Transform):
        var transform_comp = get_component(C_Transform)
        transform_comp.transform = global_transform

    # Add conditional components based on game state
    if GameState.is_multiplayer:
        add_component(C_NetworkSync.new())

    if GameState.debug_mode:
        add_component(C_DebugInfo.new())
```

### Instantiating Entity Prefabs
实例化实体预制件

**Basic Spawning Pattern:
基础生成模式：**

```gdscript
# Spawn system or main scene
@export var player_prefab: PackedScene
@export var enemy_prefab: PackedScene

func spawn_player(position: Vector3) -> Entity:
    var player = player_prefab.instantiate() as Entity
    player.global_position = position
    get_tree().current_scene.add_child(player)  # Add to scene
    ECS.world.add_entity(player)  # Register with ECS
    return player

func spawn_enemy(position: Vector3) -> Entity:
    var enemy = enemy_prefab.instantiate() as Entity
    enemy.global_position = position
    get_tree().current_scene.add_child(enemy)
    ECS.world.add_entity(enemy)
    return enemy
```

**Advanced Spawning with SpawnSystem:
使用 SpawnSystem 的高级生成：**

```gdscript
# s_spawner.gd
class_name SpawnerSystem
extends System

func query():
    return q.with_all([C_SpawnPoint])

func process(entity: Entity, delta: float):
    var spawn_point = entity.get_component(C_SpawnPoint)

    if spawn_point.should_spawn():
        var spawned = spawn_point.prefab.instantiate() as Entity
        spawned.global_position = entity.global_position
        get_tree().current_scene.add_child(spawned)
        ECS.world.add_entity(spawned)

        spawn_point.mark_spawned()
```

**Prefab Management Best Practices:
预制体管理最佳实践：**

```gdscript
# Organize prefabs in preload statements
const PLAYER_PREFAB = preload("res://entities/gameplay/e_player.tscn")
const ENEMY_PREFAB = preload("res://entities/enemies/e_enemy.tscn")
const WEAPON_PREFAB = preload("res://entities/items/e_weapon.tscn")

# Or use a prefab registry
class_name PrefabRegistry

static var prefabs = {
    "player": preload("res://entities/gameplay/e_player.tscn"),
    "enemy": preload("res://entities/enemies/e_enemy.tscn"),
    "weapon": preload("res://entities/items/e_weapon.tscn")
}

static func spawn(prefab_name: String, position: Vector3) -> Entity:
    var prefab = prefabs[prefab_name]
    var entity = prefab.instantiate() as Entity
    entity.global_position = position
    get_tree().current_scene.add_child(entity)
    ECS.world.add_entity(entity)
    return entity
```

## 🏗️ Main Scene Architecture
🏗️ 主场景架构

### Scene Structure Pattern
场景结构模式

Organize your main scene using the proven structure pattern:
使用经过验证的结构模式组织主要场景：

```
Main.tscn
├── World (World node)
├── DefaultSystems (Node - instantiated from default_systems.tscn)
│   ├── run-first (Node - SystemGroup)
│   │   ├── VictimInitSystem
│   │   └── EcsStorageLoad
│   ├── input (Node - SystemGroup)
│   │   ├── ItemSystem
│   │   ├── WeaponsSystem
│   │   └── PlayerControlsSystem
│   ├── gameplay (Node - SystemGroup)
│   │   ├── GearSystem
│   │   ├── DeathSystem
│   │   └── EventSystem
│   ├── physics (Node - SystemGroup)
│   │   ├── FrictionSystem
│   │   ├── CharacterBody3DSystem
│   │   └── TransformSystem
│   ├── ui (Node - SystemGroup)
│   │   └── UiVisibilitySystem
│   ├── debug (Node - SystemGroup)
│   │   └── DebugLabel3DSystem
│   └── run-last (Node - SystemGroup)
│       ├── ActionsSystem
│       └── PendingDeleteSystem
├── Level (Node3D - for level geometry)
└── Entities (Node3D - spawned entities go here)
```

### Systems Setup in Main Scene
主要场景中的系统设置

**Scene-based Systems Setup (Recommended)
基于场景的系统设置（推荐）**

Use scene composition to organize systems. The default\_systems.tscn contains all systems organized by execution groups:
使用场景组成来组织系统。default\_systems.tscn 包含了按执行组组织的所有系统：

```gdscript
# main.gd - Simple main scene setup
extends Node

@onready var world: World = $World

func _ready():
    Bootstrap.bootstrap()  # Initialize any game-specific setup
    ECS.world = world
    # Systems are automatically registered via scene composition
```

**Creating a Default Systems Scene:
创建默认系统场景：**

1.  Create `default_systems.tscn` with system groups as Node children
    创建 `default_systems.tscn` ，并将系统组作为 Node 的子节点
2.  Add individual system scripts as children of each group
    为每个组添加单独的系统脚本作为子节点
3.  Instantiate this scene in your main scene
    在主场景中实例化此场景
4.  Systems are automatically discovered and registered by the World
    系统会由世界自动发现并注册

### Processing Systems by Group
按组处理系统

```gdscript
# main.gd - Process systems in correct order
extends Node3D

func _process(delta):
    if ECS.world:
        ECS.process(delta, "input")     # Handle input first
        ECS.process(delta, "core")      # Core logic
        ECS.process(delta, "gameplay")  # Game mechanics
        ECS.process(delta, "render")    # UI/visual updates last

func _physics_process(delta):
    if ECS.world:
        ECS.process(delta, "physics")   # Physics systems
```

## 🛠️ Common Utility Patterns
🛠️ 常用实用程序模式

### Transform Synchronization
同步转换

Common transform synchronization patterns:
常见的转换同步模式：

```gdscript
# Sync entity transform TO component (scene → component)
static func sync_transform_to_component(entity: Entity):
    if entity.has_component(C_Transform):
        var transform_comp = entity.get_component(C_Transform)
        transform_comp.transform = entity.global_transform

# Sync component transform TO entity (component → scene)
static func sync_component_to_transform(entity: Entity):
    if entity.has_component(C_Transform):
        var transform_comp = entity.get_component(C_Transform)
        entity.global_transform = transform_comp.transform

# Common usage in entity on_ready()
func on_ready():
    sync_transform_to_component(self)  # Sync scene position to C_Transform
```

### Component Helpers
组件辅助函数

Build helpers for common component operations:
为常见组件操作构建辅助函数：

```gdscript
# Helper functions you can add to your project
static func add_health_to_entity(entity: Entity, max_health: float):
    var health = C_Health.new(max_health)
    entity.add_component(health)
    return health

static func damage_entity(entity: Entity, amount: float):
    if entity.has_component(C_Health):
        var health = entity.get_component(C_Health)
        health.current = max(0, health.current - amount)
        return health.current <= 0  # Return true if entity died
    return false
```

## 🎯 Next Steps
下一步

Now that you understand best practices:
现在你已经了解了最佳实践：

1.  **Apply these patterns** in your projects
    将这些模式应用到你的项目中
2.  **Learn advanced topics** in [Core Concepts](CORE_CONCEPTS.md)
    学习核心概念中的高级主题
3.  **Optimize performance** with [Performance Guide](PERFORMANCE_OPTIMIZATION.md)
    使用性能指南优化性能

**Need help?** [Join our Discord](https://discord.gg/eB43XU2tmn) for community discussions and support.
需要帮助？加入我们的 Discord 以参与社区讨论和获取支持。

* * *

*"Good ECS code is like a well-organized toolbox - every component has its place, every system has its purpose, and everything works together smoothly."
“良好的 ECS 代码就像一个井然有序的工具箱——每个组件都有其位置，每个系统都有其目的，一切都能顺畅地协同工作。”*