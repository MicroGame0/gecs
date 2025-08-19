# Getting Started with GECS
使用 GECS 入门

> **Build your first ECS project in 5 minutes**
> **5 分钟内构建你的第一个 ECS 项目**

This guide will walk you through creating a simple player entity with health and transform components using GECS. By the end, you'll understand the core concepts and have a working example.
本指南将带你使用 GECS 创建一个带有健康和变换组件的简单玩家实体。完成后，你将了解核心概念并拥有一个可运行的示例。

## 📋 Prerequisites
📋 先决条件

*   Godot 4.x installed
    Godot 4.x 已安装
*   Basic GDScript knowledge
    基本 GDScript 知识
*   5 minutes of your time
    您需要花费 5 分钟时间

## ⚡ Step 1: Setup (1 minute)
⚡ 第一步：设置（1 分钟）

### Install GECS
安装 GECS

1.  **Download GECS** and place it in your project's `addons/` folder
    下载 GECS 并将其放置在项目中的 `addons/` 文件夹内
2.  **Enable the plugin**: Go to `Project > Project Settings > Plugins` and enable "GECS"
    启用插件：前往 `Project > Project Settings > Plugins` 并启用 "GECS"
3.  **Verify setup**: The ECS singleton should be automatically added to AutoLoad
    验证设置：ECS 单例应自动添加到 AutoLoad 中

> 💡 **Quick Check**: If you see errors, make sure `ECS` appears in `Project > Project Settings > AutoLoad`
> 💡 快速检查：如果出现错误，请确保 `ECS` 出现在 `Project > Project Settings > AutoLoad` 中

## 🎮 Step 2: Your First Entity (1 minute)
🎮 第二步：你的第一个实体（1 分钟）

Create a new scene and script for your player entity:
创建一个新的场景和脚本，用于你的玩家实体：

**File: `e_player.gd`
文件： `e_player.gd`**

```gdscript
# e_player.gd
class_name Player
extends Entity

func on_ready():
    # Sync the entity's scene position to the Transform component
    if has_component(C_Transform):
        var transform_comp = get_component(C_Transform)
        transform_comp.position = global_position
```

> 💡 **What's happening?** Entities are containers for components. We're creating a player entity that will sync its transform with the component system.
> 💡 发生了什么？实体是组件的容器。我们正在创建一个玩家实体，该实体会与组件系统同步其变换数据。

## 📦 Step 3: Your First Components (1 minute)
📦 第三步：你的第一个组件（1 分钟）

Components hold data. Let's create health and transform components:
组件持有数据。让我们创建健康和变换组件：

**File: `c_health.gd`
文件： `c_health.gd`**

```gdscript
# c_health.gd
class_name C_Health
extends Component

@export var current: float = 100.0
@export var maximum: float = 100.0

func _init(max_health: float = 100.0):
    maximum = max_health
    current = max_health
```

**File: `c_transform.gd`
文件: `c_transform.gd`**

```gdscript
# c_transform.gd
class_name C_Transform
extends Component

@export var position: Vector3 = Vector3.ZERO

func _init(pos: Vector3 = Vector3.ZERO):
    position = pos
```

**File: `c_velocity.gd`
文件: `c_velocity.gd`**

```gdscript
# c_velocity.gd
class_name C_Velocity
extends Component

@export var velocity: Vector3 = Vector3.ZERO

func _init(vel: Vector3 = Vector3.ZERO):
    velocity = vel
```

> 💡 **Key Principle**: Components only hold data, never logic. Think of them as data containers.
> 💡 关键原则：组件只持有数据，从不包含逻辑。将它们视为数据容器。

## ⚙️ Step 4: Your First System (1 minute)
⚙️ 第四步：你的第一个系统（1 分钟）

Systems contain the logic that operates on entities with specific components. This system moves entities across the screen:
系统包含针对具有特定组件的实体进行操作的逻辑。此系统在屏幕上移动实体：

**File: `s_movement.gd`
文件： `s_movement.gd`**

```gdscript
# s_movement.gd
class_name MovementSystem
extends System

func query():
    # Find all entities that have both transform and velocity
    return q.with_all([C_Transform, C_Velocity])

func process(entity: Entity, delta: float):
    var transform_comp = entity.get_component(C_Transform)
    var velocity_comp = entity.get_component(C_Velocity)
    
    # Move the entity based on its velocity
    transform_comp.position += velocity_comp.velocity * delta
    
    # Update the actual entity position in the scene
    entity.global_position = transform_comp.position
    
    # Bounce off screen edges (simple example)
    if transform_comp.position.x > 10 or transform_comp.position.x < -10:
        velocity_comp.velocity.x *= -1
```

> 💡 **System Logic**: Query finds entities with required components, process() runs the movement logic on each entity every frame.
> 💡 系统逻辑：查询找到具有所需组件的实体，每帧在每个实体上运行移动逻辑。

## 🎬 Step 5: See It Work (1 minute)
🎬 第 5 步：查看效果（1 分钟）

Now let's put it all together in a main scene:
场景介绍日志中：

**File: `main.gd`
文件：RUN.py !**

```gdscript
# main.gd
extends Node

@onready var world: World = $World

func _ready():
    ECS.world = world
    
    # Create a moving player entity
    var player = Player.new()
    player.add_components([
        C_Health.new(100),
        C_Transform.new(),
        C_Velocity.new(Vector3(2, 0, 0))  # Move right at 2 units/second
    ])
    add_child(player)  # Add to scene tree
    ECS.world.add_entity(player)  # Add to ECS world
    
    # Create the movement system
    var movement_system = MovementSystem.new()
    ECS.world.add_system(movement_system)

func _process(delta):
    # Process all systems
    if ECS.world:
        ECS.process(delta)
```

**Run your project!** 🎉 You now have a working ECS setup where the player entity moves across the screen and bounces off the edges! The MovementSystem updates entity positions based on their velocity components.
运行成功！ ECS 设置中：玩家实体在屏幕上移动，并，碰到边缘时会弹跳！运动系统根据实体的速度更新位置。

## 🎯 What You Just Built
🎯 你构建了什么？

Congratulations! You've created your first ECS project with:
恭喜！您已创建了第一个 ECS 项目，包括：

*   **Entity**: Player - a container for components
    Entity: Player - 一个组件容器
*   **Components**: C\_Health, C\_Transform, C\_Velocity - pure data containers
    Components: C\_Health, C\_Transform, C\_Velocity - 纯数据容器
*   **System**: MovementSystem - logic that moves entities based on velocity
    System: MovementSystem - 基于速度移动实体的逻辑
*   **World**: Container that manages entities and systems
    世界：管理实体和系统的容器

## 📈 Next Steps
下一步

Now that you have the basics working, here's how to level up:
现在你已经掌握了基本知识，接下来可以提升一下：

### 1\. Create Entity Prefabs (Recommended)
1\. 创建实体预制件（推荐）

Instead of creating entities in code, use Godot's scene system:
而不是在代码中创建实体，使用 Godot 的场景系统：

1.  **Create a new scene** with your Entity class as the root node
    创建一个新的场景，将你的 Entity 类作为根节点
2.  **Add visual children** (MeshInstance3D, Sprite3D, etc.)
    添加视觉子节点（MeshInstance3D、Sprite3D 等）
3.  **Add components via define\_components()** or `component_resources` array in Inspector
    通过 define\_components()或 Inspector 中的 `component_resources` 数组添加组件
4.  **Save as .tscn file** (e.g., `e_player.tscn`)
    保存为 .tscn 文件（例如： `e_player.tscn` ）
5.  **Load and instantiate** in your main scene
    在主场景中加载并实例化

```gdscript
# Improved e_player.gd with define_components()
class_name Player
extends Entity

func define_components() -> Array:
    return [
        C_Health.new(100),
        C_Transform.new(),
        C_Velocity.new(Vector3(1, 0, 0))  # Move right slowly
    ]

func on_ready():
    # Sync scene position to component
    if has_component(C_Transform):
        var transform_comp = get_component(C_Transform)
        transform_comp.position = global_position
```

### 2\. Organize Your Main Scene
2\. 组织主场景

Structure your main scene using the proven scene-based pattern:
使用基于场景的模式结构化你的主场景：

```
Main.tscn
├── World (World node)
├── DefaultSystems (instantiated from default_systems.tscn)
│   ├── input (SystemGroup)
│   ├── gameplay (SystemGroup) 
│   ├── physics (SystemGroup)
│   └── ui (SystemGroup)
├── Level (Node3D for static environment)
└── Entities (Node3D for spawned entities)
```

**Benefits:
优势：**

*   **Visual organization** in Godot editor
    Godot 编辑器中的可视化组织
*   **Easy system reordering** between groups
    组间系统重排序便捷
*   **Reusable system configurations
    可复用的系统配置**

### 3\. Learn More Patterns
3\. 学习更多模式

### 🧠 Understand the Concepts
🧠 理解概念

**→ [Core Concepts Guide](CORE_CONCEPTS-zh-CN-dual.md)** - Deep dive into Entities, Components, Systems, and Relationships
→ 核心概念指南 - 深入了解实体、组件、系统和关系

### 🔧 Add More Features
🔧 添加更多功能

Try adding these to your moving player:
尝试为移动玩家添加以下功能：

*   **Input system** - Add C\_Input component and system to control movement with arrow keys
    输入系统 - 添加 C\_Input 组件和系统，使用方向键控制移动
*   **Multiple entities** - Create more moving objects with different velocities
    多个实体 - 创建更多具有不同速度的移动对象
*   **Collision system** - Add C\_Collision component and detect when entities hit each other
    碰撞系统 - 添加 C\_Collision 组件，并检测实体之间是否发生碰撞
*   **Gravity system** - Add downward velocity to make entities fall
    重力系统 - 添加向下速度使实体下落

### 📚 Learn Best Practices
📚 学习最佳实践

**→ [Best Practices Guide](BEST_PRACTICES-zh-CN-dual.md)** - Write maintainable ECS code
→ 最佳实践指南 - 编写可维护的 ECS 代码

### 🔧 Explore Advanced Features
🔧 探索高级功能

*   **[Component Queries](COMPONENT_QUERIES-zh-CN-dual.md)** - Filter by component property values
    组件查询 - 通过组件属性值进行过滤
*   **[Relationships](RELATIONSHIPS-zh-CN-dual.md)** - Link entities together for complex interactions
    关系 - 用于复杂交互的实体链接
*   **[Observers](OBSERVERS-zh-CN-dual.md)** - Reactive systems that respond to changes
    观察者 - 响应变化的反应式系统
*   **[Performance Optimization](PERFORMANCE_OPTIMIZATION-zh-CN-dual.md)** - Make your games run fast
    性能优化 - 让你的游戏运行得更快

## ❓ Having Issues?
❓ 有问题吗？

### Player not responding?
播放器无响应？

*   Check that `ECS.process(delta)` is called in `_process()`
    请检查 `ECS.process(delta)` 是否在 `_process()` 中被调用
*   Verify components are added to the entity via `define_components()` or Inspector
    请验证组件是否通过 `define_components()` 或者 Inspector 添加到实体中
*   Make sure the system is added to the world
    确保系统已添加到世界中
*   Ensure transform synchronization is called in entity's `on_ready()`
    确保在实体的 `on_ready()` 中调用了变换同步

### Errors in console?
控制台中有错误吗？

*   Check that all classes extend the correct base class
    检查所有类是否正确继承了基类
*   Verify file names match class names
    验证文件名是否与类名匹配
*   Ensure GECS plugin is enabled
    确保已启用 GECS 插件

**Still stuck?** → [Troubleshooting Guide](TROUBLESHOOTING-zh-CN-dual.md)
仍然卡住了？→ 故障排查指南

## 🏆 What's Next?
🏆 接下来做什么？

You're now ready to build amazing games with GECS! The Entity-Component-System pattern will help you:
你现在可以使用 GECS 构建令人惊叹的游戏了！实体-组件-系统模式将帮助你：

*   **Scale your game** - Add features without breaking existing code
    扩展你的游戏 - 在不破坏现有代码的情况下添加新功能
*   **Reuse code** - Components and systems work across different entity types
    重用代码 - 组件和系统可以在不同实体类型之间共享
*   **Debug easier** - Clear separation between data and logic
    更轻松地调试 - 数据和逻辑之间有清晰的分离
*   **Optimize performance** - GECS handles efficient querying for you
    优化性能 - GECS 会为你处理高效的查询

**Ready to dive deeper?** Start with [Core Concepts](CORE_CONCEPTS-zh-CN-dual.md) to really understand what makes ECS powerful.
准备好深入了解了吗？从核心概念开始，真正理解 ECS 的强大之处。

**Need help?** [Join our Discord community](https://discord.gg/eB43XU2tmn) for support and discussions.
需要帮助吗？加入我们的 Discord 社区获取支持和讨论。

* * *

*"The best way to learn ECS is to build with it. Start simple, then add complexity as you understand the patterns."
“学习 ECS 最好的方式是用它来构建。从简单开始，随着对模式的理解逐渐增加复杂性。”*
