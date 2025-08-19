# GECS Performance Optimization Guide
GECS 性能优化指南

> **Make your ECS games run fast and smooth
> 让你的 ECS 游戏运行快速流畅**

This guide shows you how to optimize your GECS-based games for maximum performance. Learn to identify bottlenecks, optimize queries, and design systems that scale.
本指南将教你如何优化基于 GECS 的游戏以获得最佳性能。学习如何识别瓶颈、优化查询以及设计可扩展的系统。

## 📋 Prerequisites
📋 前置条件

*   Understanding of [Core Concepts](CORE_CONCEPTS-zh-CN-dual.md)
    对核心概念的理解
*   Familiarity with [Best Practices](BEST_PRACTICES-zh-CN-dual.md)
    熟悉最佳实践
*   A working GECS project to optimize
    一个用于优化的 GECS 项目

## 🎯 Performance Fundamentals
🎯 性能基础

### The ECS Performance Model
ECS 性能模型

GECS performance depends on three key factors:
GECS 性能取决于三个关键因素：

1.  **Query Efficiency** - How fast you find entities
    查询效率 - 你查找实体的速度
2.  **Component Access** - How quickly you read/write data
    组件访问 - 你读写数据的速度
3.  **System Design** - How well your logic is organized
    系统设计 - 你的逻辑组织得有多好

Most performance gains come from optimizing these in order of impact.
大多数性能提升都来自于按影响顺序优化这些方面。

## 🔍 Profiling Your Game
🔍 分析您的游戏

### Monitor Query Cache Performance
监控查询缓存性能

Always profile before optimizing. GECS provides query cache statistics for performance monitoring:
优化前始终进行性能分析。GECS 提供查询缓存统计数据用于性能监控：

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

### Use Godot's Built-in Profiler
使用 Godot 内置的分析器

Monitor your game's performance in the Godot editor:
在 Godot 编辑器中监控游戏性能：

1.  **Run your project** in debug mode
    以调试模式运行项目
2.  **Open the Profiler** (Debug → Profiler)
    打开分析器（调试 → 分析器）
3.  **Look for ECS-related spikes** in the frame time
    在帧时间中查找与 ECS 相关的峰值
4.  **Identify the slowest systems** in your processing groups
    识别您处理组中最慢的系统

## ⚡ Query Optimization
⚡ 查询优化

### 1\. Use Proper System Query Pattern
1\. 使用正确的系统查询模式

GECS automatically handles query optimization when you follow the standard pattern:
GECS 在您遵循标准模式时自动处理查询优化：

```gdscript
# ✅ Good - Standard GECS pattern (automatically optimized)
class_name MovementSystem extends System

func query():
    return q.with_all([C_Position, C_Velocity]).with_none([C_Frozen])

func process(entity: Entity, delta: float):
    var pos = entity.get_component(C_Position)
    var vel = entity.get_component(C_Velocity)
    pos.value += vel.value * delta
```

```gdscript
# ❌ Avoid - Manual query building in process methods
func process_all(entities: Array, delta: float):
    # Don't do this - bypasses automatic query optimization
    var custom_entities = ECS.world.query.with_all([C_Position]).execute()
    # Process custom_entities...
```

### 2\. Optimize Query Specificity
2\. 优化查询的特定性

More specific queries run faster:
更具体的查询运行更快：

```gdscript
# ✅ Fast - Specific query
class_name PlayerInputSystem extends System
func query():
    return q.with_all([C_Input, C_Movement]).with_group("player")
    # Only matches 1 entity typically

# ✅ Fast - Type-specific query
class_name ProjectileSystem extends System
func query():
    return q.with_all([C_Projectile, C_Velocity])
    # Only matches projectiles
```

```gdscript
# ❌ Slow - Overly broad query
class_name UniversalSystem extends System
func query():
    return q.with_all([C_Position])
    # Matches almost everything in the game!

func process(entity: Entity, delta: float):
    # Now we need expensive type checking
    if entity.has_component(C_Player):
        # Handle player...
    elif entity.has_component(C_Enemy):
        # Handle enemy...
    # This defeats the purpose of ECS!
```

### 3\. Minimize with\_any Queries
3\. 尽量减少 with\_any 查询

`with_any` queries are slower than `with_all`. Use sparingly:
`with_any` 查询比 `with_all` 慢。请谨慎使用：

```gdscript
# ✅ Better - Split into specific systems
class_name PlayerMovementSystem extends System
func query(): return q.with_all([C_Player, C_Movement])

class_name EnemyMovementSystem extends System
func query(): return q.with_all([C_Enemy, C_Movement])
```

```gdscript
# ❌ Slower - Generic with_any query
class_name MovementSystem extends System
func query():
    return q.with_all([C_Movement])
          .with_any([C_Player, C_Enemy])
    # This has to check both component types for every entity
```

## 🧱 Component Design for Performance
🧱 组件设计以提高性能

### Keep Components Lightweight
保持组件轻量化

Smaller components = faster memory access:
较小的组件 = 更快的内存访问：

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

### Minimize Component Additions/Removals
最小化组件的添加/删除

Adding and removing components requires index updates. Batch component operations when possible:
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

### Choose Between Boolean Properties vs Components Based on Usage
根据使用情况选择布尔属性还是组件

The choice between boolean properties and separate components depends on how frequently states change and how many entities need them.
选择布尔属性还是单独组件取决于状态变化的频率以及需要它们的实体数量。

#### Use Boolean Properties for Frequently-Changing States
对于经常变化的状态使用布尔属性

When states change often, boolean properties avoid expensive index updates:
当状态经常变化时，布尔属性避免昂贵的索引更新：

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

**Tradeoffs:
权衡：**

*   ✅ Fast state changes (no index rebuilds)
    ✅ 快速状态变化（无需重建索引）
*   ✅ Simple property checks in systems
    ✅ 系统中简单的属性检查
*   ❌ All entities need the state component (memory overhead)
    ❌ 所有实体都需要状态组件（内存开销）
*   ❌ Less precise queries (can't easily find "only stunned entities")
    ❌ 查询不够精确（无法轻松找到"仅被眩晕的实体"）

#### Use Separate Components for Rare or Permanent States
为稀有或永久状态使用单独组件

When states are long-lasting or infrequent, separate components provide precise queries:
当状态持续时间长或不常见时，单独组件提供精确查询：

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

**Tradeoffs:
权衡：**

*   ✅ Memory efficient (only entities with states have components)
    ✅ 内存高效（只有具有状态的实体才有组件）
*   ✅ Precise queries for specific states
    ✅ 精确查询特定状态
*   ❌ State changes trigger expensive index updates
    ❌ 状态变化会触发昂贵的索引更新
*   ❌ Complex queries with multiple exclusions
    ❌ 包含多个排除条件的复杂查询

#### Guidelines:
指南：

*   **High-frequency changes** (every few frames): Use boolean properties
    高频变化（每几帧一次）：使用布尔属性
*   **Low-frequency changes** (minutes apart): Use separate components
    低频变化（几分钟一次）：使用单独的组件
*   **Related states** (buffs/debuffs): Group into property components
    相关状态（增益/减益）：按属性组件分组
*   **Distinct entity types** (player/enemy): Use separate components
    不同实体类型（玩家/敌人）：使用独立组件

## ⚙️ System Performance Patterns
⚙️ 系统性能模式

### Early Exit Strategies
早期退出策略

Return early when no processing is needed:
当无需处理时尽早返回：

```gdscript
class_name HealthRegenerationSystem extends System

func process(entity: Entity, delta: float):
    var health = entity.get_component(C_Health)

    # Early exits for common cases
    if health.current >= health.maximum:
        return  # Already at full health

    if health.regeneration_rate <= 0:
        return  # No regeneration configured

    # Only do expensive work when needed
    health.current = min(health.current + health.regeneration_rate * delta, health.maximum)
```

### Batch Entity Operations
批量实体操作

Group entity operations together:
将实体操作组合在一起：

```gdscript
# ✅ Good - Batch creation
func spawn_enemy_wave():
    var enemies: Array[Entity] = []

    # Create all entities using entity pooling
    for i in range(50):
        var enemy = ECS.world.create_entity()  # Uses entity pool for performance
        setup_enemy_components(enemy)
        enemies.append(enemy)

    # Add all to world at once
    ECS.world.add_entities(enemies)

# ✅ Good - Individual removal (batch removal not available)
func cleanup_dead_entities():
    var dead_entities = ECS.world.query.with_all([C_Dead]).execute()
    for entity in dead_entities:
        ECS.world.remove_entity(entity)  # Remove individually
```

## 📊 Performance Targets
📊 性能目标

### Frame Rate Targets
帧率目标

Aim for these processing times per frame:
每帧处理时间目标：

*   **60 FPS target**: ECS processing < 16ms per frame
    60 FPS 目标：ECS 处理 < 16ms 每帧
*   **30 FPS target**: ECS processing < 33ms per frame
    30 FPS 目标：ECS 处理 < 33ms 每帧
*   **Mobile target**: ECS processing < 8ms per frame
    移动目标：ECS 处理 < 每帧 8 毫秒

### Entity Scale Guidelines
实体规模指南

GECS handles these entity counts well with proper optimization:
GECS 在适当优化下能很好地处理这些实体数量：

*   **Small games**: 100-500 entities
    小型游戏：100-500 个实体
*   **Medium games**: 500-2000 entities
    中规游戏：500-2000 个实体
*   **Large games**: 2000-10000 entities
    大型游戏：2000-10000 个实体
*   **Massive games**: 10000+ entities (requires advanced optimization)
    超大型游戏：10000+个实体（需要高级优化）

## 🎯 Next Steps
🎯 下一步

1.  **Profile your current game** to establish baseline performance
    分析您当前游戏以建立基准性能
2.  **Apply query optimizations** from this guide
    应用本指南中的查询优化
3.  **Redesign heavy components** into lighter, focused ones
    将重型组件重新设计为更轻量、更专注的组件
4.  **Implement system improvements** like early exits and batching
    实现系统改进，如提前退出和批处理
5.  **Consider advanced techniques** like pooling and spatial partitioning for demanding scenarios
    考虑在要求高的场景中使用池化和空间划分等高级技术

## 🔍 Additional Performance Features
🔍 额外的性能功能

### Entity Pooling
实体池

GECS includes built-in entity pooling for optimal performance:
GECS 包含内置的实体池以实现最佳性能：

```gdscript
# Use the entity pool for frequent entity creation/destruction
var new_entity = ECS.world.create_entity()  # Gets from pool when available
```

### Query Cache Statistics
查询缓存统计

Monitor query performance with built-in cache tracking:
使用内置缓存跟踪监控查询性能：

```gdscript
# Get detailed cache performance data
var stats = ECS.world.get_cache_stats()
print("Cache hit rate: ", stats.get("hits", 0) / (stats.get("hits", 0) + stats.get("misses", 1)))
```

**Need more help?** Check the [Troubleshooting Guide](TROUBLESHOOTING-zh-CN-dual.md) for specific performance issues.
需要更多帮助？请查看故障排除指南以解决特定的性能问题。

* * *

*"Fast ECS code isn't about clever tricks - it's about designing systems that naturally align with how the framework works best."
"快速 ECS 代码不在于巧妙的技巧，而在于设计那些自然符合框架最佳工作方式的系统。"*
