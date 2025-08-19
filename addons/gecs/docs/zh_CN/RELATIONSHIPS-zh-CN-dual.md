# Relationships in GECS
GECS 中的关系

> **Link entities together for complex game interactions
> 链接实体以实现复杂的游戏交互**

Relationships allow you to connect entities in meaningful ways, creating dynamic associations that go beyond simple component data. This guide shows you how to use GECS's relationship system to build complex game mechanics.
关系允许你以有意义的方式连接实体，创建超越简单组件数据的动态关联。本指南将向你展示如何使用 GECS 的关系系统来构建复杂的游戏机制。

## 📋 Prerequisites
📋 前置条件

*   Understanding of [Core Concepts](CORE_CONCEPTS-zh-CN-dual.md)
    对核心概念的理解
*   Familiarity with [Query System](CORE_CONCEPTS-zh-CN-dual.md#query-system)
    熟悉查询系统

## 🔗 What are Relationships?
🔗 什么是关系？

Think of **components** as the data that makes up an entity's state, and **relationships** as the links that connect entity to entity. Relationships can be simple links or carry data about the connection itself.
将组件视为构成实体状态的 数据，而关系则是连接实体与实体的链接。关系可以是简单的链接，也可以携带有关连接本身的数据。

In GECS, relationships consist of three parts:
在 GECS 中，关系由三个部分组成：

*   **Source** - Entity that has the relationship (e.g., Bob)
    源 - 拥有关系的实体（例如，鲍勃）
*   **Relation** - Component defining the relationship type (e.g., "Likes")
    关系 - 定义关系类型的组件（例如，“喜欢”）
*   **Target** - Entity or type being related to (e.g., Alice)
    目标 - 被关联的实体或类型（例如，Alice）

This creates powerful queries like "find all entities that like Alice" or "find all enemies attacking the player."
这可以创建强大的查询，例如“查找所有喜欢 Alice 的实体”或“查找所有攻击玩家的敌人。”

## 🎯 Core Relationship Concepts
🎯 核心关系概念

### Relationship Components
关系组件

Relationships use components to define their type and can carry data:
关系使用组件来定义其类型，并可以携带数据：

```gdscript
# c_likes.gd - Simple relationship
class_name C_Likes
extends Component

# c_loves.gd - Another simple relationship
class_name C_Loves
extends Component

# c_eats.gd - Relationship with data
class_name C_Eats
extends Component

@export var quantity: int = 1

func _init(qty: int = 1):
    quantity = qty
```

### Creating Relationships
创建关系

```gdscript
# Create entities
var e_bob = Entity.new()
var e_alice = Entity.new()
var e_heather = Entity.new()
var e_apple = Food.new()

# Add to world
ECS.world.add_entity(e_bob)
ECS.world.add_entity(e_alice)
ECS.world.add_entity(e_heather)
ECS.world.add_entity(e_apple)

# Create relationships
e_bob.add_relationship(Relationship.new(C_Likes.new(), e_alice))        # bob likes alice
e_alice.add_relationship(Relationship.new(C_Loves.new(), e_heather))    # alice loves heather
e_heather.add_relationship(Relationship.new(C_Likes.new(), Food))       # heather likes food (type)
e_heather.add_relationship(Relationship.new(C_Eats.new(5), e_apple))    # heather eats 5 apples

# Remove relationships
e_alice.remove_relationship(Relationship.new(C_Loves.new(), e_heather)) # alice no longer loves heather
```

## 🔍 Relationship Queries
🔍 关系查询

### Basic Relationship Queries
基本关系查询

**Query for Specific Relationships:
特定关系查询：**

```gdscript
# Any entity that likes alice
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), e_alice)])

# Any entity that eats 5 apples
ECS.world.query.with_relationship([Relationship.new(C_Eats.new(5), e_apple)])

# Any entity that likes the Food entity type
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), Food)])
```

**Exclude Relationships:
排除关系：**

```gdscript
# Entities with any relation toward heather that don't like bob
ECS.world.query
    .with_relationship([Relationship.new(ECS.wildcard, e_heather)])
    .without_relationship([Relationship.new(C_Likes.new(), e_bob)])
```

### Wildcard Relationships
通配符关系

Use `ECS.wildcard` (or `null`) to query for any relation or target:
使用 `ECS.wildcard` （或 `null` ）查询任何关系或目标：

```gdscript
# Any entity with any relation toward heather
ECS.world.query.with_relationship([Relationship.new(ECS.wildcard, e_heather)])

# Any entity that likes anything
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), ECS.wildcard)])
ECS.world.query.with_relationship([Relationship.new(C_Likes.new())])  # Omitting target = wildcard

# Any entity with any relation to Enemy entity type
ECS.world.query.with_relationship([Relationship.new(ECS.wildcard, Enemy)])
```

### Reverse Relationships
反向关系

Find entities that are the **target** of relationships:
查找作为关系目标实体的实例：

```gdscript
# Find entities that are being liked by someone
ECS.world.query.with_reverse_relationship([Relationship.new(C_Likes.new(), ECS.wildcard)])

# Find entities being attacked
ECS.world.query.with_reverse_relationship([Relationship.new(C_IsAttacking.new())])

# Find food being eaten
ECS.world.query.with_reverse_relationship([Relationship.new(C_Eats.new(), ECS.wildcard)])
```

## 🎮 Game Examples
🎮 游戏示例

### Combat System with Relationships
战斗系统与关系

```gdscript
# Combat relationship components
class_name C_IsAttacking extends Component
@export var damage: float = 10.0

class_name C_IsTargeting extends Component
class_name C_IsAlliedWith extends Component

# Create combat entities
var player = Player.new()
var enemy1 = Enemy.new()
var enemy2 = Enemy.new()
var ally = Ally.new()

# Setup relationships
enemy1.add_relationship(Relationship.new(C_IsAttacking.new(25.0), player))
enemy2.add_relationship(Relationship.new(C_IsTargeting.new(), player))
player.add_relationship(Relationship.new(C_IsAlliedWith.new(), ally))

# Combat system queries
class_name CombatSystem extends System

func get_entities_attacking_player():
    var player = get_player_entity()
    return ECS.world.query.with_relationship([
        Relationship.new(C_IsAttacking.new(), player)
    ]).execute()

func get_player_allies():
    var player = get_player_entity()
    return ECS.world.query.with_reverse_relationship([
        Relationship.new(C_IsAlliedWith.new(), player)
    ]).execute()
```

### Hierarchical Entity System
层级实体系统

```gdscript
# Hierarchy relationship components
class_name C_ParentOf extends Component
class_name C_ChildOf extends Component
class_name C_OwnerOf extends Component

# Create hierarchy
var parent = Entity.new()
var child1 = Entity.new()
var child2 = Entity.new()
var weapon = Weapon.new()

# Setup parent-child relationships
parent.add_relationship(Relationship.new(C_ParentOf.new(), child1))
parent.add_relationship(Relationship.new(C_ParentOf.new(), child2))
child1.add_relationship(Relationship.new(C_ChildOf.new(), parent))
child2.add_relationship(Relationship.new(C_ChildOf.new(), parent))

# Setup ownership
child1.add_relationship(Relationship.new(C_OwnerOf.new(), weapon))

# Hierarchy system queries
class_name HierarchySystem extends System

func get_children_of_entity(entity: Entity):
    return ECS.world.query.with_relationship([
        Relationship.new(C_ParentOf.new(), entity)
    ]).execute()

func get_parent_of_entity(entity: Entity):
    return ECS.world.query.with_reverse_relationship([
        Relationship.new(C_ParentOf.new(), entity)
    ]).execute()
```

## 🏗️ Relationship Best Practices
🏗️ 关系最佳实践

### Performance Optimization
性能优化

**Reuse Relationship Objects:
重用关系对象：**

```gdscript
# ✅ Good - Reuse for performance
var r_likes_apples = Relationship.new(C_Likes.new(), e_apple)
var r_attacking_players = Relationship.new(C_IsAttacking.new(), Player)

# Use the same relationship object multiple times
entity1.add_relationship(r_attacking_players)
entity2.add_relationship(r_attacking_players)
```

**Static Relationship Factory (Recommended):
静态关系工厂（推荐）：**

```gdscript
# ✅ Excellent - Organized relationship management
class_name Relationships

static func attacking_players():
    return Relationship.new(C_IsAttacking.new(), Player)

static func attacking_anything():
    return Relationship.new(C_IsAttacking.new(), ECS.wildcard)

static func chasing_players():
    return Relationship.new(C_IsChasing.new(), Player)

static func interacting_with_anything():
    return Relationship.new(C_Interacting.new(), ECS.wildcard)

static func equipped_on_anything():
    return Relationship.new(C_EquippedOn.new(), ECS.wildcard)

# Usage in systems:
var attackers = ECS.world.query.with_relationship([Relationships.attacking_players()]).execute()
var chasers = ECS.world.query.with_relationship([Relationships.chasing_anything()]).execute()
```

### Naming Conventions
命名规范

**Relationship Components:
关系组件：**

*   Use descriptive names that clearly indicate the relationship
    使用描述性名称，清晰表明关系
*   Follow the `C_VerbNoun` pattern when possible
    尽可能遵循 `C_VerbNoun` 模式
*   Examples: `C_Likes`, `C_IsAttacking`, `C_OwnerOf`, `C_MemberOf`
    示例： `C_Likes` , `C_IsAttacking` , `C_OwnerOf` , `C_MemberOf`

**Relationship Variables:
关系变量：**

*   Use `r_` prefix for relationship instances
    使用 `r_` 前缀表示关系实例
*   Examples: `r_likes_alice`, `r_attacking_player`, `r_parent_of_child`
    示例： `r_likes_alice` , `r_attacking_player` , `r_parent_of_child`

## 🎯 Next Steps
🎯 下一步

Now that you understand relationships:
现在你已经理解了关系：

1.  **Design relationship schemas** for your game's entities
    为你的游戏实体设计关系模式
2.  **Experiment with wildcard queries** for dynamic systems
    尝试使用通配符查询动态系统
3.  **Combine relationships with component queries** for powerful filtering
    结合关系与组件查询实现强大过滤
4.  **Optimize with static relationship factories** for better performance
    使用静态关系工厂优化性能
5.  **Learn advanced patterns** in [Best Practices Guide](BEST_PRACTICES-zh-CN-dual.md)
    在最佳实践指南中学习高级模式

## 📚 Related Documentation
📚 相关文档

*   **[Core Concepts](CORE_CONCEPTS-zh-CN-dual.md)** - Understanding the ECS fundamentals
    核心概念 - 理解 ECS 基础
*   **[Component Queries](COMPONENT_QUERIES-zh-CN-dual.md)** - Advanced property-based filtering
    组件查询 - 高级属性过滤
*   **[Best Practices](BEST_PRACTICES-zh-CN-dual.md)** - Write maintainable ECS code
    最佳实践 - 编写可维护的 ECS 代码
*   **[Performance Optimization](PERFORMANCE_OPTIMIZATION-zh-CN-dual.md)** - Optimize relationship queries
    性能优化 - 优化关系查询

* * *

*"Relationships turn a collection of entities into a living, interconnected game world where entities can react to each other in meaningful ways."
"关系将一组实体转变为一个生动、相互连接的游戏世界，其中实体可以以有意义的方式相互反应。"*
