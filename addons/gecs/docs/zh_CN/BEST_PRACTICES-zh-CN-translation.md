# GECS 最佳实践指南

> **编写可维护、高性能的 ECS 代码**

本指南涵盖了使用 GECS 构建健壮游戏的有效模式和最佳实践。应用这些模式以保持代码的清洁、快速和易于调试。

## 📋 前置条件

*   完成[入门指南](GETTING_STARTED-zh-CN-translation.md)
*   理解[核心概念](CORE_CONCEPTS-zh-CN-translation.md)

## 🧱 组件设计模式

### 保持组件纯净数据

组件应仅持有数据，绝不应包含逻辑或行为。

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

### 使用组合优于继承

通过组合简单组件来构建实体，而不是复杂的继承层次结构。

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

### 面向配置进行设计

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

## ⚙️ 系统设计模式

### 单一职责原则

每个系统应处理一个特定关注点。

```gdscript
# ✅ Good - Focused systems
class_name MovementSystem extends System
func query(): return q.with_all([C_Position, C_Velocity])

class_name RenderSystem extends System
func query(): return q.with_all([C_Position, C_Sprite])

class_name HealthSystem extends System
func query(): return q.with_all([C_Health])
```

### 使用系统组进行处理顺序

使用基于场景的组织方式将系统组织成逻辑组。系统在场景节点中分组，并按正确的顺序进行处理。

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

### 为性能优化而提前退出

当无需工作时，从系统处理中提前返回。

```gdscript
# ✅ Good - Early exit patterns
class_name HealthRegenerationSystem extends System

func query():
    return q.with_all([C_Health]).with_none([C_Dead])

func process(entities: Array[Entity], components: Array, delta: float):
    for entity in entities:
        var health = entity.get_component(C_Health)

        # Early exit if already at max health
        if health.current >= health.maximum:
            continue

        # Apply regeneration
        health.current = min(health.current + health.regeneration_rate * delta, health.maximum)
```

### 使用 CommandBuffer 在迭代期间进行结构变更

在系统处理期间添加/移除组件、实体或关系时，使用 `cmd` CommandBuffer 而不是直接调用世界/实体。这允许安全的正向迭代和延迟缓存失效。

```gdscript
# ✅ Good - Use CommandBuffer for safe iteration
class_name LifetimeSystem extends System

func query():
    return q.with_all([C_Lifetime])

func process(entities: Array[Entity], components: Array, delta: float):
    for entity in entities:
        var lifetime = entity.get_component(C_Lifetime)
        lifetime.time -= delta
        if lifetime.time <= 0:
            cmd.remove_entity(entity)  # Queued, executed after system completes
```

```gdscript
# ❌ Avoid - Direct removal during iteration requires backwards iteration
func process(entities: Array[Entity], components: Array, delta: float):
    for i in range(entities.size() - 1, -1, -1):
        if should_delete(entities[i]):
            ECS.world.remove_entity(entities[i])  # Modifies array during iteration
```

刷新模式控制排队命令执行的时间：

*   PER\_SYSTEM（默认）— 每个系统完成后执行
*   PER\_GROUP — 在组内所有系统完成后执行
*   MANUAL — 需要显式 `ECS.world.flush_command_buffers()` 调用

## 🏗️ 代码组织模式

### GECS 命名规范

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

### 文件组织

按主题组织您的 ECS 文件以实现更好的可扩展性：

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

## 🎮 常见游戏模式

### 玩家角色模式

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

### 敌对模式

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

## 🚀 性能最佳实践

### 选择正确的查询方法 ⭐ 新功能！

查询性能排名 (v5.0.0-rc4+)：

```gdscript
# 🏆 FASTEST - Enabled/disabled queries (constant time)
class_name ActiveEntitiesOnly extends System
func query():
    return q.enabled(true)  # ~0.05ms for any number of entities

# 🥈 EXCELLENT - Component queries (heavily optimized)
class_name MovementSystem extends System
func query():
    return q.with_all([C_Position, C_Velocity])  # ~0.6ms for 10K entities

# 🥉 GOOD - Use with_any strategically
class_name DamageableSystem extends System
func query():
    return q.with_any([C_Player, C_Enemy]).with_all([C_Health])  # ~5.6ms for 10K

# 🐌 AVOID - Group queries are slowest
class_name PlayerSystem extends System
func query():
    return q.with_group("player")  # ~16ms for 10K entities
    # Better: q.with_all([C_Player])
```

### 使用 iterate() 进行批量性能优化

```gdscript
# ✅ Good - Batch processing with iterate()
class_name TransformSystem
extends System

func query():
    # Use iterate() to get component arrays
    return q.with_all([C_Transform]).iterate([C_Transform])

func process(entities: Array[Entity], components: Array, delta: float):
    # Batch access to components for better performance
    var transforms = components[0]  # C_Transform array from iterate()
    for i in range(entities.size()):
        entities[i].global_transform = transforms[i].transform
```

### 使用特定查询

```gdscript
# ✅ BEST - Combine enabled filter with components
class_name ActivePlayerInputSystem extends System
func query():
    return q.with_all([C_Input, C_Movement]).enabled(true)
    # Super fast: enabled filtering + component matching

# ✅ GOOD - Specific component query
class_name ProjectileSystem extends System
func query():
    return q.with_all([C_Projectile, C_Velocity])  # Fast and specific

# ❌ AVOID - Group-based queries (slow)
class_name PlayerSystem extends System
func query():
    return q.with_group("player")  # Use q.with_all([C_Player]) instead

# ❌ AVOID - Overly broad queries
class_name UniversalMovementSystem extends System
func query():
    return q.with_all([C_Transform])  # Too broad - matches everything
```

## 🎭 实体预制体（场景文件）

### 使用 Godot 场景作为实体预制体

GECS 中最强大的模式是使用 Godot 的场景系统（.tscn 文件）作为实体预制件。这结合了 ECS 数据和 Godot 的可视化编辑器：

```
e_player.tscn Structure:
├── Player (Entity node - extends your e_player.gd class)
│   ├── MeshInstance3D (visual representation)
│   ├── CollisionShape3D (physics collision)
│   ├── AudioStreamPlayer3D (sound effects)
│   └── SkeletonAttachment3D (for equipment)
```

**基于场景的预制件的优势：**

*   可视化编辑：在 Godot 的 3D 编辑器中设计实体
*   组件分配：在检查器中设置 ECS 组件
*   Godot 集成：利用现有的 Godot 节点和系统
*   可重用性：多次实例化相同的预制件
*   版本控制：场景文件与 git 配合良好

**设置实体预制件：**

1.  创建以实体为根的场景： `e_player.tscn` ，包含 `Player` 实体节点。
    *   另一个技巧是添加 CharacterBody3d，然后通过这种方式扩展 CharacterBody3D，使用 e\_player.gd 脚本，这样你就能获得 Entity 类和 CharacterBody3D 类的数据。
2.  添加视觉/物理子节点：将 MeshInstance3D、CollisionShape3D 等作为子节点添加
3.  在检查器中配置组件：将组件添加到 `component_resources` 数组中
4.  保存为可重复使用的预制件：保存.tscn 文件以进行实例化
5.  设置 on\_ready()：处理任何初始化逻辑

### 预制件中的组件分配

**方法 1：检视器分配（推荐）**

直接在 Godot Inspector 中设置组件：

```gdscript
# In e_player.tscn entity root node Inspector:
# Component Resources array:
# - [0] C_Health.new() (max: 100, current: 100)
# - [1] C_Transform.new() (synced with scene transform)
# - [2] C_Input.new() (for player controls)
# - [3] C_LocalPlayer.new() (mark as local player)
```

**方法 2：define\_components()（编程式）**

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

**方法 3：混合方法**

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

### 实例化实体预制件

**基本生成模式：**

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

**使用 SpawnSystem 的高级生成：**

```gdscript
# s_spawner.gd
class_name SpawnerSystem
extends System

func query():
    return q.with_all([C_SpawnPoint])

func process(entities: Array[Entity], components: Array, delta: float):
    for entity in entities:
        var spawn_point = entity.get_component(C_SpawnPoint)

        if spawn_point.should_spawn():
            var spawned = spawn_point.prefab.instantiate() as Entity
            spawned.global_position = entity.global_position
            get_tree().current_scene.add_child(spawned)
            ECS.world.add_entity(spawned)

            spawn_point.mark_spawned()
```

**预制体管理最佳实践：**

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

## 🏗️ 主场景架构

### 场景结构模式

使用经过验证的结构模式组织你的主场景：

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

### 主场景中的系统设置

**基于场景的系统设置（推荐）**

使用场景组合来组织系统。default\_systems.tscn 包含按执行组组织的所有系统：

```gdscript
# main.gd - Simple main scene setup
extends Node

@onready var world: World = $World

func _ready():
    Bootstrap.bootstrap()  # Initialize any game-specific setup
    ECS.world = world
    # Systems are automatically registered via scene composition
```

**创建默认系统场景：**

1.  创建 `default_systems.tscn` 并将系统组作为 Node 的子节点
2.  将各个系统脚本作为每个组的子节点添加
3.  在你的主场景中实例化这个场景
4.  系统由世界自动发现和注册

### 按组处理系统

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

## 🛠️ 常见工具模式

### 转换同步

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

### 组件辅助工具

为常见组件操作构建辅助工具：

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

## 🎛️ 关系管理最佳实践

### 有限的移除模式

**使用描述性常量：**

```gdscript
# ✅ Good - Clear intent with constants
const WEAK_CLEANSE = 1
const MEDIUM_CLEANSE = 3
const STRONG_CLEANSE = -1  # All

# ✅ Good - Stack-based constants
const SINGLE_STACK = 1
const PARTIAL_STACKS = 3
const ALL_STACKS = -1

func cleanse_debuffs(entity: Entity, power: int):
    match power:
        1: entity.remove_relationship(Relations.any_debuff(), WEAK_CLEANSE)
        2: entity.remove_relationship(Relations.any_debuff(), MEDIUM_CLEANSE)
        3: entity.remove_relationship(Relations.any_debuff(), STRONG_CLEANSE)
```

**移除前验证：**

```gdscript
# ✅ Excellent - Safe removal with validation
func safe_partial_heal(entity: Entity, heal_amount: int):
    var damage_rels = entity.get_relationships(Relations.any_damage())
    if damage_rels.is_empty():
        print("Entity has no damage to heal")
        return

    var to_heal = min(heal_amount, damage_rels.size())
    entity.remove_relationship(Relations.any_damage(), to_heal)
    print("Healed ", to_heal, " damage effects")

# ✅ Good - Helper function with built-in safety
func remove_poison_stacks(entity: Entity, stacks_to_remove: int):
    if stacks_to_remove <= 0:
        return
    entity.remove_relationship(Relations.poison_effect(), stacks_to_remove)
```

**系统集成模式：**

```gdscript
# ✅ Excellent - Integration with game systems
class_name StatusEffectSystem extends System

func process(entities: Array[Entity], components: Array, delta: float):
    # Example: process spell casting entities
    for entity in entities:
        var spell = entity.get_component(C_SpellCaster)
        if spell.is_casting_cleanse():
            process_cleanse_spell(entity, spell.target, spell.power)

func process_cleanse_spell(caster: Entity, target: Entity, spell_power: int):
    # Calculate cleanse strength based on spell power and caster stats
    var cleanse_strength = calculate_cleanse_strength(caster, spell_power)

    # Apply graduated cleansing based on strength
    match cleanse_strength:
        1..3:   target.remove_relationship(Relations.any_debuff(), 1)
        4..6:   target.remove_relationship(Relations.any_debuff(), 2)
        7..9:   target.remove_relationship(Relations.any_debuff(), 3)
        _:      target.remove_relationship(Relations.any_debuff())  # Remove all

func process_antidote_item(user: Entity, antidote_strength: int):
    # Remove poison based on antidote quality
    user.remove_relationship(Relations.poison_effect(), antidote_strength)

    # Remove poison resistance temporarily to prevent immediate repoison
    user.add_relationship(Relations.poison_immunity(), 5.0)  # 5 second immunity

class_name InventorySystem extends System

func consume_item_stack(entity: Entity, item_type: Script, count: int):
    # Consume specific number of items from inventory
    entity.remove_relationship(
        Relationship.new(C_HasItem.new(), item_type),
        count
    )

func use_consumable(entity: Entity, item: Component, quantity: int = 1):
    # Use consumable items with quantity
    entity.remove_relationship(
        Relationship.new(C_HasItem.new(), item),
        quantity
    )
```

**性能优化：**

```gdscript
# ✅ Good - Cache relationships for multiple operations
func optimize_bulk_removal(entity: Entity):
    # Cache the relationship for reuse
    var poison_rel = Relations.poison_effect()
    var damage_rel = Relations.any_damage()

    # Multiple targeted removals
    entity.remove_relationship(poison_rel, 2)      # Remove 2 poison
    entity.remove_relationship(damage_rel, 1)      # Remove 1 damage
    entity.remove_relationship(poison_rel, 1)      # Remove 1 more poison

# ✅ Excellent - Batch removal patterns
func batch_cleanup(entities: Array[Entity]):
    var cleanup_rel = Relations.temporary_effect()

    for entity in entities:
        # Remove up to 3 temporary effects from each entity
        entity.remove_relationship(cleanup_rel, 3)
```

## 🎯 下一步

现在你已经了解了最佳实践：

1.  将这些模式应用于您的项目
2.  学习核心概念中的高级主题
3.  使用性能指南优化性能

需要帮助？加入我们的 Discord 社区讨论和支持。

* * *

*好的 ECS 代码就像一个组织良好的工具箱——每个组件都有其位置，每个系统都有其目的，所有东西都能顺利协同工作。*