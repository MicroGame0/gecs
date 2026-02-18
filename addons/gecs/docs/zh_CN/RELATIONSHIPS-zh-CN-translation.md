# GECS 中的关系

> **链接实体以实现复杂的游戏交互**

关系允许你以有意义的方式连接实体，创建超越简单组件数据的动态关联。本指南将向你展示如何使用 GECS 的关系系统来构建复杂的游戏机制。

## 📋 前置条件

*   理解[核心概念](CORE_CONCEPTS-zh-CN-translation.md)
*   熟悉[查询系统](CORE_CONCEPTS-zh-CN-translation.md#query-system)

## 🔗 什么是关系？

将 **组件** 视为构成实体状态的数据，将 **关系** 视为连接实体与其他实体、组件或类型的链接。关系可以是简单的链接，也可以携带有关连接本身的数据。

在 GECS 中，关系由三个部分组成：

*   **源** \- 拥有关系的实体（例如，Bob）
*   **关系** \- 定义关系类型的组件（例如，“喜欢”，“损坏”）
*   **目标** \- 被关联的对象：实体、组件实例或原型（例如，Alice，FireDamage 组件，Enemy 类）

## 🎯 关系类型

GECS 支持三种强大的关系模式：

### 1\. **实体关系**

将实体链接到其他实体：

```gdscript
# Bob likes Alice (entity to entity)
e_bob.add_relationship(Relationship.new(C_Likes.new(), e_alice))
```

### 2\. **组件关系**

将实体链接到组件实例以建立类型层次结构：

```gdscript
# Entity has fire damage (entity to component)
entity.add_relationship(Relationship.new(C_Damaged.new(), C_FireDamage.new(50)))
```

### 3\. **原型关系**

将实体链接到类/类型：

```gdscript
# Heather likes all food (entity to type)
e_heather.add_relationship(Relationship.new(C_Likes.new(), Food))
```

这可以创建强大的查询，例如“查找所有喜欢 Alice 的实体”、“查找所有受到火损伤的实体”或“查找所有受到任何损伤的实体”。

## 🎯 核心关系概念

### 关系组件

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

### 创建关系

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

# Remove with limits (NEW)
e_player.remove_relationship(Relationship.new(C_Poison.new(), null), 1)  # Remove only 1 poison stack
e_enemy.remove_relationship(Relationship.new(C_Buff.new(), null), 3)     # Remove up to 3 buffs
e_hero.remove_relationship(Relationship.new(C_Damage.new(), null), -1)   # Remove all damage (default)
```

## 🔍 关系查询

### 基本关系查询

**特定关系查询：**

```gdscript
# Any entity that likes alice (type matching)
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), e_alice)])

# Any entity that eats apples (type matching)
ECS.world.query.with_relationship([Relationship.new(C_Eats.new(), e_apple)])

# Any entity that eats 5 or more apples (component query)
ECS.world.query.with_relationship([
    Relationship.new({C_Eats: {'quantity': {"_gte": 5}}}, e_apple)
])

# Any entity that likes the Food entity type
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), Food)])
```

**排除关系：**

```gdscript
# Entities with any relation toward heather that don't like bob
ECS.world.query
    .with_relationship([Relationship.new(ECS.wildcard, e_heather)])
    .without_relationship([Relationship.new(C_Likes.new(), e_bob)])
```

### 通配符关系

使用 `ECS.wildcard`（或 `null`）查询任何关系或目标：

```gdscript
# Any entity with any relation toward heather
ECS.world.query.with_relationship([Relationship.new(ECS.wildcard, e_heather)])

# Any entity that likes anything
ECS.world.query.with_relationship([Relationship.new(C_Likes.new(), ECS.wildcard)])
ECS.world.query.with_relationship([Relationship.new(C_Likes.new())])  # Omitting target = wildcard

# Any entity with any relation to Enemy entity type
ECS.world.query.with_relationship([Relationship.new(ECS.wildcard, Enemy)])
```

### 基于组件的关系

将实体链接到 **组件实例** 以实现强大的类型层次结构和数据系统：

```gdscript
# Damage system using component targets
class_name C_Damaged extends Component
class_name C_FireDamage extends Component
    @export var amount: int = 0
    func _init(dmg: int = 0): amount = dmg

class_name C_PoisonDamage extends Component
    @export var amount: int = 0
    func _init(dmg: int = 0): amount = dmg

# Entity has multiple damage types
entity.add_relationship(Relationship.new(C_Damaged.new(), C_FireDamage.new(50)))
entity.add_relationship(Relationship.new(C_Damaged.new(), C_PoisonDamage.new(25)))

# Query for entities with any damage type (wildcard)
var damaged_entities = ECS.world.query.with_relationship([
    Relationship.new(C_Damaged.new(), null)
]).execute()

# Query for entities with fire damage >= 50 using component query
var high_fire_damaged = ECS.world.query.with_relationship([
    Relationship.new(C_Damaged.new(), {C_FireDamage: {"amount": {"_gte": 50}}})
]).execute()

# Query for entities with any fire damage (type matching)
var any_fire_damaged = ECS.world.query.with_relationship([
    Relationship.new(C_Damaged.new(), C_FireDamage)
]).execute()
```

### 匹配模式

GECS 关系支持两种匹配模式：

#### 类型匹配（默认）

通过组件类型匹配关系，忽略属性值：

```gdscript
# Matches any C_Damaged relationship regardless of amount
entity.has_relationship(Relationship.new(C_Damaged.new(), target))

# Matches any fire damage effect by type
entity.has_relationship(Relationship.new(C_Damaged.new(), C_FireDamage.new()))

# Query for any entities with fire damage (type matching)
var any_fire_damaged = ECS.world.query.with_relationship([
    Relationship.new(C_Damaged.new(), C_FireDamage)
]).execute()
```

#### 组件查询匹配

通过特定属性标准使用字典匹配关系：

```gdscript
# Match C_Damaged relationships where amount >= 50
var high_damage = ECS.world.query.with_relationship([
    Relationship.new({C_Damaged: {'amount': {"_gte": 50}}}, target)
]).execute()

# Match fire damage with specific duration
var lasting_fire = ECS.world.query.with_relationship([
    Relationship.new(
        C_Damaged.new(),
        {C_FireDamage: {'duration': {"_gt": 5.0}}}
    )
]).execute()

# Match both relation AND target with queries
var strong_buffs = ECS.world.query.with_relationship([
    Relationship.new(
        {C_Buff: {'duration': {"_gt": 10}}},
        {C_Player: {'level': {"_gte": 5}}}
    )
]).execute()
```

**何时使用每种：**

*   **类型匹配** ：查找具有"任何火灾损伤"、"任何此类增益"的实体
*   **组件查询** ：查找具有确切损伤数值、特定增益持续时间或属性标准的实体

### 关系中的组件查询

使用字典通过特定属性值查询关系：

```gdscript
# Query by relation property
var heavy_eaters = ECS.world.query.with_relationship([
    Relationship.new({C_Eats: {'amount': {"_gte": 5}}}, e_apple)
]).execute()

# Query by target component property
var high_hp_targets = ECS.world.query.with_relationship([
    Relationship.new(C_Targeting.new(), {C_Health: {'hp': {"_gte": 100}}})
]).execute()

# Query operators: _eq, _ne, _gt, _lt, _gte, _lte, _in, _nin, func
var special_damage = ECS.world.query.with_relationship([
    Relationship.new(
        {C_Damage: {'type': {"_in": ["fire", "ice"]}}},
        null
    )
]).execute()

# Complex multi-property queries
var critical_effects = ECS.world.query.with_relationship([
    Relationship.new(
        {C_Effect: {
            'damage': {"_gt": 20},
            'duration': {"_gte": 10.0},
            'type': {"_eq": "critical"}
        }},
        null
    )
]).execute()
```

### 反向关系

查找作为关系**目标**的实体：

```gdscript
# Find entities that are being liked by someone
ECS.world.query.with_reverse_relationship([Relationship.new(C_Likes.new(), ECS.wildcard)])

# Find entities being attacked
ECS.world.query.with_reverse_relationship([Relationship.new(C_IsAttacking.new())])

# Find food being eaten
ECS.world.query.with_reverse_relationship([Relationship.new(C_Eats.new(), ECS.wildcard)])
```

## 🎛️ 限制关系移除

> **精确控制要移除的关系数量以实现细粒度管理**

`remove_relationship()` 方法现在支持一个 **限制参数** ，允许你精确控制要移除的匹配关系数量。这对于基于栈的系统、部分修复、库存管理和细粒度效果控制至关重要。

### 基本语法

```gdscript
entity.remove_relationship(relationship, limit)
```

**限制值：**

*   `limit = -1`（默认值）：移除**所有**匹配关系
*   `limit = 0`：不移除**任何**关系（用于测试/验证）
*   `limit = 1`：移除**一个**匹配关系
*   `limit > 1`: 移除**最多该数量**的匹配关系

### 核心使用场景

#### 1\. **基于堆栈的系统**

非常适合增益/减益堆栈、持续伤害效果或任何效果可以堆叠的系统：

```gdscript
# Poison stack system
class_name C_PoisonStack extends Component
@export var damage_per_tick: float = 5.0

# Apply poison stacks
entity.add_relationship(Relationship.new(C_PoisonStack.new(3.0), null))
entity.add_relationship(Relationship.new(C_PoisonStack.new(3.0), null))
entity.add_relationship(Relationship.new(C_PoisonStack.new(3.0), null))
entity.add_relationship(Relationship.new(C_PoisonStack.new(3.0), null))  # 4 poison stacks

# Antidote removes 2 poison stacks
entity.remove_relationship(Relationship.new(C_PoisonStack.new(), null), 2)
# Entity now has 2 poison stacks remaining

# Strong antidote removes all poison
entity.remove_relationship(Relationship.new(C_PoisonStack.new(), null))  # Default: remove all
```

#### 2\. **部分治疗系统**

控制伤害移除以实现渐进式治疗或部分修复：

```gdscript
# Multiple damage sources on entity
entity.add_relationship(Relationship.new(C_Damage.new(), C_FireDamage.new(25)))
entity.add_relationship(Relationship.new(C_Damage.new(), C_FireDamage.new(15)))
entity.add_relationship(Relationship.new(C_Damage.new(), C_SlashDamage.new(30)))
entity.add_relationship(Relationship.new(C_Damage.new(), C_PoisonDamage.new(10)))

# Healing potion removes one damage source
entity.remove_relationship(Relationship.new(C_Damage.new(), null), 1)

# Fire resistance removes only fire damage (up to 2 sources)
entity.remove_relationship(Relationship.new(C_Damage.new(), C_FireDamage), 2)

# Full heal removes all damage
entity.remove_relationship(Relationship.new(C_Damage.new(), null))  # All damage gone
```

#### 3\. **物品和资源管理**

处理物品堆叠、资源消耗和制作材料：

```gdscript
# Item stack system
class_name C_HasItem extends Component
class_name C_HealthPotion extends Component
@export var healing_amount: int = 50

# Player has multiple health potions
entity.add_relationship(Relationship.new(C_HasItem.new(), C_HealthPotion.new(50)))
entity.add_relationship(Relationship.new(C_HasItem.new(), C_HealthPotion.new(50)))
entity.add_relationship(Relationship.new(C_HasItem.new(), C_HealthPotion.new(50)))
entity.add_relationship(Relationship.new(C_HasItem.new(), C_HealthPotion.new(50)))

# Use one health potion
entity.remove_relationship(Relationship.new(C_HasItem.new(), C_HealthPotion), 1)

# Vendor buys 2 health potions
entity.remove_relationship(Relationship.new(C_HasItem.new(), C_HealthPotion), 2)

# Drop all potions
entity.remove_relationship(Relationship.new(C_HasItem.new(), C_HealthPotion))
```

#### 4\. **增益/减益管理**

对临时效果进行精细控制：

```gdscript
# Multiple speed buffs from different sources
entity.add_relationship(Relationship.new(C_Buff.new(), C_SpeedBuff.new(1.2, 10.0)))  # Boots
entity.add_relationship(Relationship.new(C_Buff.new(), C_SpeedBuff.new(1.5, 5.0)))   # Spell
entity.add_relationship(Relationship.new(C_Buff.new(), C_SpeedBuff.new(1.1, 30.0)))  # Passive

# Dispel magic removes one buff
entity.remove_relationship(Relationship.new(C_Buff.new(), null), 1)

# Mass dispel removes up to 3 buffs
entity.remove_relationship(Relationship.new(C_Buff.new(), null), 3)

# Purge removes all buffs
entity.remove_relationship(Relationship.new(C_Buff.new(), null))
```

### 高级示例

#### 组件查询 + 限制组合

结合组件查询与限制以实现精确控制：

```gdscript
# Remove only high-damage effects (damage > 20), up to 2 of them
entity.remove_relationship(
    Relationship.new({C_Damage: {"amount": {"_gt": 20}}}, null),
    2
)

# Remove poison effects with duration < 5 seconds, limit to 1
entity.remove_relationship(
    Relationship.new({C_PoisonEffect: {"duration": {"_lt": 5.0}}}, null),
    1
)

# Remove fire damage with specific amount range, up to 3 instances
entity.remove_relationship(
    Relationship.new(
        C_Damage.new(),
        {C_FireDamage: {"amount": {"_gte": 10, "_lte": 50}}}
    ),
    3
)

# Remove all fire damage regardless of amount (no limit, type matching)
entity.remove_relationship(
    Relationship.new(C_Damage.new(), C_FireDamage),
    -1
)

# Remove buffs with specific multiplier, limit to 2
entity.remove_relationship(
    Relationship.new({C_Buff: {"multiplier": {"_gte": 1.5}}}, null),
    2
)
```

#### 系统集成

将有限移除集成到您的游戏系统中：

```gdscript
class_name HealingSystem extends System

func heal_entity(entity: Entity, healing_power: int):
    """Remove damage based on healing power"""
    if healing_power <= 0:
        return
    
    # Partial healing - remove damage effects based on healing power
    var damage_to_remove = min(healing_power, get_damage_count(entity))
    entity.remove_relationship(Relationship.new(C_Damage.new(), null), damage_to_remove)
    
    print("Healed ", damage_to_remove, " damage effects")

func get_damage_count(entity: Entity) -> int:
    return entity.get_relationships(Relationship.new(C_Damage.new(), null)).size()

class_name CleanseSystem extends System

func cleanse_entity(entity: Entity, cleanse_strength: int):
    """Remove debuffs based on cleanse strength"""
    match cleanse_strength:
        1:  # Weak cleanse
            entity.remove_relationship(Relationship.new(C_Debuff.new(), null), 1)
        2:  # Medium cleanse  
            entity.remove_relationship(Relationship.new(C_Debuff.new(), null), 3)
        3:  # Strong cleanse
            entity.remove_relationship(Relationship.new(C_Debuff.new(), null))  # All debuffs

class_name CraftingSystem extends System

func consume_materials(entity: Entity, recipe: Dictionary):
    """Consume specific amounts of crafting materials"""
    for material_type in recipe:
        var amount_needed = recipe[material_type]
        entity.remove_relationship(
            Relationship.new(C_HasMaterial.new(), material_type), 
            amount_needed
        )
```

### 错误处理与验证

限制参数提供内置安全措施：

```gdscript
# Safe operations - won't crash if fewer relationships exist than requested
entity.remove_relationship(Relationship.new(C_Buff.new(), null), 100)  # Removes all available, won't error

# Validation operations
entity.remove_relationship(Relationship.new(C_Damage.new(), null), 0)  # Removes nothing - useful for testing

# Check before removal
var damage_count = entity.get_relationships(Relationship.new(C_Damage.new(), null)).size()
if damage_count > 0:
    entity.remove_relationship(Relationship.new(C_Damage.new(), null), min(3, damage_count))
```

### 性能考虑

有限移除针对效率进行了优化：

```gdscript
# ✅ Efficient - stops searching after finding enough matches
entity.remove_relationship(Relationship.new(C_Effect.new(), null), 5)

# ✅ Still efficient - reuses the same removal logic
entity.remove_relationship(Relationship.new(C_Effect.new(), null), -1)  # Remove all

# ✅ Most efficient for single removals
entity.remove_relationship(Relationship.new(C_SpecificEffect.new(exact_data), target), 1)
```

### 与多重关系集成

与 `remove_relationships()` 无缝配合进行批量操作：

```gdscript
# Apply limit to multiple relationship types
var relationships_to_remove = [
    Relationship.new(C_Buff.new(), null),
    Relationship.new(C_Debuff.new(), null),
    Relationship.new(C_TemporaryEffect.new(), null)
]

# Remove up to 2 of each type
entity.remove_relationships(relationships_to_remove, 2)
```

## 🎮 游戏示例

### 组件关系状态效果系统

本示例展示了如何使用基于组件的关系构建灵活的状态效果系统：

```gdscript
# Status effect marker
class_name C_HasEffect extends Component

# Damage type components
class_name C_FireDamage extends Component
    @export var damage_per_second: float = 10.0
    @export var duration: float = 5.0
    func _init(dps: float = 10.0, dur: float = 5.0):
        damage_per_second = dps
        duration = dur

class_name C_PoisonDamage extends Component
    @export var damage_per_tick: float = 5.0
    @export var ticks_remaining: int = 10
    func _init(dpt: float = 5.0, ticks: int = 10):
        damage_per_tick = dpt
        ticks_remaining = ticks

# Buff type components  
class_name C_SpeedBuff extends Component
    @export var multiplier: float = 1.5
    @export var duration: float = 10.0
    func _init(mult: float = 1.5, dur: float = 10.0):
        multiplier = mult
        duration = dur

class_name C_StrengthBuff extends Component
    @export var bonus_damage: float = 25.0
    @export var duration: float = 8.0
    func _init(bonus: float = 25.0, dur: float = 8.0):
        bonus_damage = bonus
        duration = dur

# Apply various effects to entities
func apply_status_effects():
    # Player gets fire damage and speed buff
    player.add_relationship(Relationship.new(C_HasEffect.new(), C_FireDamage.new(15.0, 8.0)))
    player.add_relationship(Relationship.new(C_HasEffect.new(), C_SpeedBuff.new(2.0, 12.0)))
    
    # Enemy gets poison and strength buff
    enemy.add_relationship(Relationship.new(C_HasEffect.new(), C_PoisonDamage.new(8.0, 15)))
    enemy.add_relationship(Relationship.new(C_HasEffect.new(), C_StrengthBuff.new(30.0, 10.0)))

# Status effect processing system
class_name StatusEffectSystem extends System

func query():
    # Get all entities with any status effects
    return ECS.world.query.with_relationship([Relationship.new(C_HasEffect.new(), null)])

func process_fire_damage():
    # Find entities with any fire damage effect (type matching)
    var fire_damaged = ECS.world.query.with_relationship([
        Relationship.new(C_HasEffect.new(), C_FireDamage)
    ]).execute()

    for entity in fire_damaged:
        # Get the actual fire damage data using type matching
        var fire_rel = entity.get_relationship(
            Relationship.new(C_HasEffect.new(), C_FireDamage.new())
        )
        var fire_damage = fire_rel.target as C_FireDamage

        # Apply damage
        apply_damage(entity, fire_damage.damage_per_second * delta)

        # Reduce duration
        fire_damage.duration -= delta
        if fire_damage.duration <= 0:
            entity.remove_relationship(fire_rel)

func process_speed_buffs():
    # Find entities with speed buffs using type matching
    var speed_buffed = ECS.world.query.with_relationship([
        Relationship.new(C_HasEffect.new(), C_SpeedBuff)
    ]).execute()

    for entity in speed_buffed:
        # Get actual speed buff data using type matching
        var speed_rel = entity.get_relationship(
            Relationship.new(C_HasEffect.new(), C_SpeedBuff.new())
        )
        var speed_buff = speed_rel.target as C_SpeedBuff

        # Apply speed modification
        apply_speed_modifier(entity, speed_buff.multiplier)

        # Handle duration
        speed_buff.duration -= delta
        if speed_buff.duration <= 0:
            entity.remove_relationship(speed_rel)

func remove_all_effects_from_entity(entity: Entity):
    # Remove all status effects using wildcard
    entity.remove_relationship(Relationship.new(C_HasEffect.new(), null))

func remove_some_effects_from_entity(entity: Entity, count: int):
    # Remove a specific number of status effects using limit parameter
    entity.remove_relationship(Relationship.new(C_HasEffect.new(), null), count)

func cleanse_one_debuff(entity: Entity):
    # Remove just one debuff (useful for cleanse spells)
    entity.remove_relationship(Relationship.new(C_Debuff.new(), null), 1)

func dispel_magic(entity: Entity, power: int):
    # Dispel magic spell removes buffs based on power level
    match power:
        1: entity.remove_relationship(Relationship.new(C_HasEffect.new(), C_SpeedBuff), 1)    # Weak dispel - 1 speed buff
        2: entity.remove_relationship(Relationship.new(C_HasEffect.new(), null), 2)          # Medium dispel - 2 any effects  
        3: entity.remove_relationship(Relationship.new(C_HasEffect.new(), null))             # Strong dispel - all effects

func antidote_healing(entity: Entity, antidote_strength: int):
    # Antidote removes poison effects based on strength
    entity.remove_relationship(Relationship.new(C_HasEffect.new(), C_PoisonDamage), antidote_strength)

func partial_fire_immunity(entity: Entity):
    # Fire immunity spell removes up to 3 fire damage effects
    entity.remove_relationship(Relationship.new(C_HasEffect.new(), C_FireDamage), 3)

func get_entities_with_damage_effects():
    # Get entities with any damage type effect (fire or poison)
    var fire_damaged = ECS.world.query.with_relationship([
        Relationship.new(C_HasEffect.new(), C_FireDamage)
    ]).execute()
    
    var poison_damaged = ECS.world.query.with_relationship([
        Relationship.new(C_HasEffect.new(), C_PoisonDamage)
    ]).execute()
    
    # Combine results
    var all_damaged = {}
    for entity in fire_damaged:
        all_damaged[entity] = true
    for entity in poison_damaged:
        all_damaged[entity] = true
        
    return all_damaged.keys()
```

### 战斗系统与关系

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

func get_high_damage_attackers():
    var player = get_player_entity()
    # Find entities attacking player with damage >= 20
    return ECS.world.query.with_relationship([
        Relationship.new({C_IsAttacking: {'damage': {"_gte": 20.0}}}, player)
    ]).execute()

func get_player_allies():
    var player = get_player_entity()
    return ECS.world.query.with_reverse_relationship([
        Relationship.new(C_IsAlliedWith.new(), player)
    ]).execute()
```

### 层级实体系统

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

## 🏗️ 关系最佳实践

### 性能优化

**重用关系对象：**

```gdscript
# ✅ Good - Reuse for performance
var r_likes_apples = Relationship.new(C_Likes.new(), e_apple)
var r_attacking_players = Relationship.new(C_IsAttacking.new(), Player)

# Use the same relationship object multiple times
entity1.add_relationship(r_attacking_players)
entity2.add_relationship(r_attacking_players)
```

**静态关系工厂（推荐）：**

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

static func any_status_effect():
    return Relationship.new(C_HasEffect.new(), null)

static func any_damage_effect():
    return Relationship.new(C_Damage.new(), null)

static func any_buff():
    return Relationship.new(C_Buff.new(), null)

# Usage in systems:
var attackers = ECS.world.query.with_relationship([Relationships.attacking_players()]).execute()
var chasers = ECS.world.query.with_relationship([Relationships.chasing_anything()]).execute()

# Usage with limits:
entity.remove_relationship(Relationships.any_status_effect(), 1)  # Remove one effect
entity.remove_relationship(Relationships.any_damage_effect(), 3)  # Remove up to 3 damage effects
entity.remove_relationship(Relationships.any_buff())              # Remove all buffs
```

**有限移除最佳实践：**

```gdscript
# ✅ Good - Clear intent with descriptive variables
var WEAK_CLEANSE = 1
var MEDIUM_CLEANSE = 3
var STRONG_CLEANSE = -1  # All

entity.remove_relationship(Relationships.any_debuff(), WEAK_CLEANSE)

# ✅ Good - Helper functions for common operations
func remove_one_poison(entity: Entity):
    entity.remove_relationship(Relationship.new(C_Poison.new(), null), 1)

func remove_all_fire_damage(entity: Entity):
    entity.remove_relationship(Relationship.new(C_Damage.new(), C_FireDamage))

func partial_heal(entity: Entity, healing_power: int):
    entity.remove_relationship(Relationship.new(C_Damage.new(), null), healing_power)

# ✅ Excellent - Validation before removal
func safe_remove_effects(entity: Entity, count: int):
    var current_effects = entity.get_relationships(Relationship.new(C_Effect.new(), null)).size()
    var to_remove = min(count, current_effects)
    if to_remove > 0:
        entity.remove_relationship(Relationship.new(C_Effect.new(), null), to_remove)
        print("Removed ", to_remove, " effects")
```

### 命名规范

**关系组件：**

*   使用描述性名称，清晰表明关系
*   尽可能遵循 `C_VerbNoun` 模式
*   示例：`C_Likes`、`C_IsAttacking`、`C_OwnerOf`、`C_MemberOf`

**关系变量：**

*   为关系实例使用 `r_` 前缀
*   例如：`r_likes_alice`、`r_attacking_player`、`r_parent_of_child`

## 🎯 下一步

现在你已经理解了关系、组件查询和有限移除：

1.  **为游戏实体设计关系模式**
2.  **尝试使用通配符查询**进行动态系统
3.  **使用组件查询**按属性标准筛选关系
4.  **为基于栈和渐进式系统实现有限移除**
5.  **将类型匹配与组件查询结合**以实现灵活的筛选
6.  **使用静态关系工厂优化**以提升性能
7.  **使用限制参数**以在治疗、制作和效果系统中实现精细控制
8.  **在[最佳实践指南中学习高级模式](BEST_PRACTICES-zh-CN-translation.md)**

**组件查询快速入门清单：**

*   ✅ 尝试基本组件查询： `Relationship.new({C_Damage: {'amount': {"_gt": 10}}}, null)`
*   ✅ 使用查询运算符：`_eq`、`_ne`、`_gt`、`_lt`、`_gte`、`_lte`、`_in`、`_nin`
*   ✅ 查询关系和目标属性
*   ✅ 使用通配符组合查询以实现灵活过滤
*   ✅ 使用类型匹配处理"此类型中的任何组件"查询

**有限移除的快速入门清单：**

*   ✅ 尝试基本限制语法： `entity.remove_relationship(rel, 1)`
*   ✅ 构建一个简单的堆栈系统（增益效果、减益效果或伤害）
*   ✅ 为常见的移除模式创建辅助函数
*   ✅ 将限制条件整合到你的游戏系统中（治疗、净化等）
*   ✅ 测试边界情况（限制 > 可用关系）
*   ✅ 结合组件查询与限制以实现精确控制

## 📚 相关文档

*   **[核心概念](CORE_CONCEPTS-zh-CN-translation.md)** \- 理解 ECS 基础
*   **[组件查询](COMPONENT_QUERIES-zh-CN-translation.md)** \- 基于属性的高级过滤
*   **[最佳实践](BEST_PRACTICES-zh-CN-translation.md)** \- 编写可维护的 ECS 代码
*   **[性能优化](PERFORMANCE_OPTIMIZATION-zh-CN-translation.md)** \- 优化关系查询

* * *

*"关系将一组实体转变为一个生动、相互连接的游戏世界，在这个世界中，实体可以以有意义的方式相互反应。"*