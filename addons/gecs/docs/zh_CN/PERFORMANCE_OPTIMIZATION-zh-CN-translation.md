# GECS 性能优化指南

> **让你的 ECS 游戏运行快速流畅**

本指南将教你如何优化基于 GECS 的游戏以获得最佳性能。学习如何识别瓶颈、优化查询以及设计可扩展的系统。

## 📋 前置条件

*   理解[核心概念](CORE_CONCEPTS-zh-CN-translation.md)
*   熟悉 [最佳实践](BEST_PRACTICES-zh-CN-translation.md)
*   一个用于优化的 GECS 项目

## 🎯 性能基础

### ECS 性能模型

GECS 性能取决于三个关键因素：

1.  **查询效率** \- 你查找实体的速度
2.  **组件访问** \- 你读写数据的速度
3.  **系统设计** \- 你的逻辑组织得有多好

大多数性能提升都来自于按影响顺序优化这些方面。

## 🔍 分析您的游戏

### 监控查询缓存性能

优化前务必进行性能分析。GECS 提供查询缓存统计数据用于性能监控：

```gdscript
# Main.gd
func _process(delta):
    ECS.process(delta)

    # Print cache performance stats every second
    if Engine.get_process_frames() % 60 == 0:
        var cache_stats = ECS.world.get_cache_stats()
        print("ECS Performance:")
        print("  Query cache hits: ", cache_stats.get("hits", 0))
        print("  Query cache misses: ", cache_stats.get("misses", 0))
        print("  Total entities: ", ECS.world.entities.size())

        # Reset stats for next measurement period
        ECS.world.reset_cache_stats()
```

### 使用 Godot 内置的分析器

在 Godot 编辑器中监控游戏性能：

1.  **以调试模式运行项目**
2.  **打开分析器** (调试 → 分析器)
3.  **在帧时间中查找与 ECS 相关的峰值**
4.  **识别处理组中最慢的系统**

## ⚡ 查询优化

### 1\. 选择正确的查询方法 ⭐ 新功能！

**自 v5.0.0-rc4 版本起** ，查询性能排名（10,000 个实体）：

1.  **`.enabled(true/false)` 查询** : **~0.05ms** 🏆 **(最快 - 尽可能使用！)**
2.  **`.with_all([Components])` 查询** : **~0.6ms** 🥈 **(大多数用例的绝佳选择)**
3.  **`.with_any([Components])` 查询** : **~5.6ms** 🥉 **(适合 OR 风格查询)**
4.  **`.with_group("name")` 查询** : **~16ms** 🐌 **(避免用于性能关键代码)**

**性能建议：**

```gdscript
# 🏆 FASTEST - Use enabled/disabled queries when you only need active entities
class_name ActiveSystemsOnly extends System
func query():
    return q.enabled(true)  # Constant-time O(1) performance!

# 🥈 EXCELLENT - Component-based queries (heavily optimized cache)
class_name MovementSystem extends System
func query():
    return q.with_all([C_Position, C_Velocity])  # ~0.6ms for 10K entities

# 🥉 GOOD - Use with_any sparingly, split into multiple systems when possible
class_name DamageableSystem extends System
func query():
    return q.with_any([C_Player, C_Enemy]).with_all([C_Health])

# 🐌 AVOID - Group queries are the slowest
class_name PlayerSystem extends System
func query():
    return q.with_group("player")  # Consider using components instead
    # Better: q.with_all([C_Player])
```

### 2\. 使用正确的系统查询模式

当您遵循标准模式时，GECS 会自动处理查询优化：

### 2\. 使用正确的系统查询模式

GECS 在您遵循标准模式时自动处理查询优化：

```gdscript
# ✅ Good - Standard GECS pattern (automatically optimized)
class_name MovementSystem extends System

func query():
    return q.with_all([C_Position, C_Velocity]).with_none([C_Frozen])

func process(entities: Array[Entity], components: Array, delta: float):
    # Process each entity
    for entity in entities:
        var pos = entity.get_component(C_Position)
        var vel = entity.get_component(C_Velocity)
        pos.value += vel.value * delta
```

```gdscript
# ❌ Avoid - Manual query building in process methods
func process(entities: Array[Entity], components: Array, delta: float):
    # Don't do this - bypasses automatic query optimization
    var custom_entities = ECS.world.query.with_all([C_Position]).execute()
    # Process custom_entities...
```

### 3\. 优化查询的特定性

更具体的查询运行更快：

```gdscript
# ✅ Fast - Use enabled filter for active entities only
class_name PlayerInputSystem extends System
func query():
    return q.with_all([C_Input, C_Movement]).enabled(true)
    # Super fast enabled filtering + component matching

# ✅ Fast - Specific component query
class_name ProjectileSystem extends System
func query():
    return q.with_all([C_Projectile, C_Velocity])
    # Only matches projectiles - very specific
```

```gdscript
# ❌ Slow - Overly broad query
class_name UniversalSystem extends System
func query():
    return q.with_all([C_Position])
    # Matches almost everything in the game!

func process(entities: Array[Entity], components: Array, delta: float):
    # Now we need expensive type checking in a loop
    for entity in entities:
        if entity.has_component(C_Player):
            # Handle player...
        elif entity.has_component(C_Enemy):
            # Handle enemy...
        # This defeats the purpose of ECS!
```

### 4\. 智能使用 with\_any 查询

`with_any` 查询比之前 **快得多** ，但仍然比 `with_all` 慢。请策略性地使用：

```gdscript
# ✅ Good - with_any for legitimate OR scenarios
class_name DamageSystem extends System
func query():
    return q.with_any([C_Player, C_Enemy, C_NPC]).with_all([C_Health])
    # When you truly need "any of these types with health"

# ✅ Better - Split when entities have different behavior
class_name PlayerMovementSystem extends System
func query(): return q.with_all([C_Player, C_Movement])

class_name EnemyMovementSystem extends System
func query(): return q.with_all([C_Enemy, C_Movement])
# Split systems = simpler logic + better performance
```

### 5\. 避免在性能关键代码中使用分组查询

分组查询现在是最慢的选项。请改用基于组件的查询：

```gdscript
# ❌ Slow - Group-based query (~16ms for 10K entities)
class_name PlayerSystem extends System
func query():
    return q.with_group("player")

# ✅ Fast - Component-based query (~0.6ms for 10K entities)
class_name PlayerSystem extends System
func query():
    return q.with_all([C_Player])
```

## 🧱 基于组件的性能设计

### 保持组件轻量

组件越小 = 更快的内存访问：

```gdscript
# ✅ Good - Lightweight components
class_name C_Position extends Component
@export var position: Vector2

class_name C_Velocity extends Component
@export var velocity: Vector2

class_name C_Health extends Component
@export var current: float
@export var maximum: float
```

```gdscript
# ❌ Heavy - Bloated component
class_name MegaComponent extends Component
@export var position: Vector2
@export var velocity: Vector2
@export var health: float
@export var mana: float
@export var inventory: Array[Item] = []
@export var abilities: Array[Ability] = []
@export var dialogue_history: Array[String] = []
# Too much data in one place!
```

### 最小化组件的添加/删除

添加和删除组件需要更新索引。尽可能批量操作组件：

```gdscript
# ✅ Good - Batch component operations
func setup_new_enemy(entity: Entity):
    # Add multiple components in one batch
    entity.add_components([
        C_Health.new(),
        C_Position.new(),
        C_Velocity.new(),
        C_Enemy.new()
    ])

# ✅ Good - Single component change when needed
func apply_damage(entity: Entity, damage: float):
    var health = entity.get_component(C_Health)
    health.current = clamp(health.current - damage, 0, health.maximum)

    if health.current <= 0:
        entity.add_component(C_Dead.new())  # Single component addition
```

### 根据使用情况选择布尔属性还是组件

布尔属性与独立组件之间的选择取决于状态变化的频率以及需要它们的实体数量。

#### 用于频繁变化的状态

当状态经常变化时，布尔属性可以避免昂贵的索引更新：

```gdscript
# ✅ Good for frequently-changing states (buffs, status effects, etc.)
class_name C_EntityState extends Component
@export var is_stunned: bool = false
@export var is_invisible: bool = false
@export var is_invulnerable: bool = false

class_name MovementSystem extends System
func query():
    return q.with_all([C_Position, C_Velocity, C_EntityState])
    # All entities that might need states must have this component

func process(entity: Entity, delta: float):
    var state = entity.get_component(C_EntityState)
    if state.is_stunned:
        return  # Just a property check - no index updates

    # Process movement...
```

**权衡：**

*   ✅ 快速状态变更（无需重建索引）
*   ✅ 系统中简单的属性检查
*   ❌ 所有实体都需要状态组件（内存开销）
*   ❌ 精确度较低的查询（难以找到"仅被击晕的实体"）

#### 为稀有或永久状态使用独立组件

当状态持续时间长或出现频率低时，独立组件可提供精确查询：

```gdscript
# ✅ Good for rare/permanent states (player vs enemy, permanent abilities)
class_name MovementSystem extends System
func query():
    return q.with_all([C_Position, C_Velocity]).with_none([C_Paralyzed])
    # Precise query - only entities that can move

# Separate systems can target specific states precisely
class_name ParalyzedSystem extends System
func query():
    return q.with_all([C_Paralyzed])  # Only paralyzed entities
```

**权衡：**

*   ✅ 内存高效（仅状态实体有组件）
*   ✅ 精准查询特定状态
*   ❌ 状态变更触发昂贵的索引更新
*   ❌ 带多重排除的复杂查询

#### 指南：

*   **高频变化** （每几帧）：使用布尔属性
*   **低频变化** （几分钟间隔）：使用独立组件
*   **相关状态** （增益/减益）：分组为属性组件
*   **不同的实体类型** (玩家/敌人): 使用独立的组件

## ⚙️ 系统性能模式

### 早期退出策略

在无需处理时提前返回:

```gdscript
class_name HealthRegenerationSystem extends System

func process(entities: Array[Entity], components: Array, delta: float):
    for entity in entities:
        var health = entity.get_component(C_Health)

        # Early exits for common cases
        if health.current >= health.maximum:
            continue  # Already at full health

        if health.regeneration_rate <= 0:
            continue  # No regeneration configured

        # Only do expensive work when needed
        health.current = min(health.current + health.regeneration_rate * delta, health.maximum)
```

### 批量实体操作

将实体操作分组。使用 CommandBuffer 进行延迟执行，并使用单一缓存失效：

```gdscript
# ✅ Best - Use CommandBuffer in systems for bulk operations
class_name CleanupSystem extends System

func query():
    return q.with_all([C_Dead])

func process(entities: Array[Entity], components: Array, delta: float):
    for entity in entities:
        cmd.remove_entity(entity)  # Queued, single cache invalidation when flushed

# ✅ Good - Batch creation outside of systems
func spawn_enemy_wave():
    var enemies: Array[Entity] = []
    for i in range(50):
        var enemy = ECS.world.create_entity()
        setup_enemy_components(enemy)
        enemies.append(enemy)
    ECS.world.add_entities(enemies)
```

**CommandBuffer 刷新模式**用于性能调优：

*   **PER\_SYSTEM**（默认）—安全，每个系统后刷新
*   **PER\_GROUP** — 将一组系统批量处理，在最后一次性刷新
*   **MANUAL** — 最大批量处理，需要显式 `ECS.world.flush_command_buffers()` 调用

## 📊 性能目标

### 帧率目标

目标每帧的处理时间：

*   **60 FPS 目标** ：ECS 处理 < 16ms 每帧
*   **30 FPS 目标** ：ECS 处理 < 33ms 每帧
*   **移动设备目标** ：ECS 处理 < 8ms 每帧

### 实体规模指南

GECS 通过适当的优化能够很好地处理这些实体数量：

*   **小型游戏** ：100-500 个实体
*   **中型游戏** ：500-2000 个实体
*   **大型游戏** : 2000-10000 个实体
*   **超大型游戏** : 10000+个实体（需要高级优化）

## 🎯 下一步

1.  **分析当前游戏**以建立基准性能
2.  **应用本指南中的查询优化**
3.  **重新设计重型组件**为更轻量、专注的组件
4.  **实现系统改进** ，如早期退出和批处理
5.  **考虑高级技术** ，如池化和空间划分，用于高要求场景

## 🔍 其他性能特性

### 实体池

GECS 包含内置实体池以实现最佳性能：

```gdscript
# Use the entity pool for frequent entity creation/destruction
var new_entity = ECS.world.create_entity()  # Gets from pool when available
```

### 查询缓存统计

使用内置缓存跟踪监控查询性能：

```gdscript
# Get detailed cache performance data
var stats = ECS.world.get_cache_stats()
print("Cache hit rate: ", stats.get("hits", 0) / (stats.get("hits", 0) + stats.get("misses", 1)))
```

**需要更多帮助吗？** 查看具有特定性能问题的 [故障排除指南](TROUBLESHOOTING-zh-CN-translation.md) 。

* * *

*"快速 ECS 代码不在于巧妙的技巧，而在于设计那些自然符合框架最佳工作方式的系统。"*