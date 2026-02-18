# 开始使用 GECS

> **5 分钟内构建你的第一个 ECS 项目**

本指南将带你通过使用 GECS 创建一个简单的玩家实体，该实体具有生命值和变换组件。到结束时，你将理解核心概念并拥有一个可运行的示例。

## 📋 前置条件

*   已安装 Godot 4.x
*   基础 GDScript 知识
*   5分钟的时间

## ⚡ 第1步：设置（1分钟）

### 安装 GECS

1.  **下载 GECS** 并将其放置在您项目的 `addons/` 文件夹中
2.  **启用插件** ：前往 `Project > Project Settings > Plugins` 并启用 "GECS"
3.  **验证设置** ：ECS 单例应该会自动添加到 AutoLoad

> 💡 **快速检查** ：如果你看到错误，请确保 `ECS` 出现在 `Project > Project Settings > AutoLoad`

## 🎮 第 2 步：你的第一个实体（2 分钟）

GECS 中的实体扩展了 Godot 的 `Node` 类。创建实体有两种方式：

### **选项 A：基于场景的实体** （用于空间属性）

当你需要访问 `Node3D` 或 `Node2D` 属性（如位置、旋转、缩放）或想添加视觉子节点（精灵、网格等）时，请使用此选项。

> ⚠️ **关键点** ：`Entity` 扩展的是 `Node`（而不是 `Node3D` 或 `Node2D`），因此请创建一个以适当的空间节点类型为根的场景，然后将你的实体脚本附加到它上面。

**步骤：**

1.  在 Godot 中 **创建新场景** ：
    
    *   点击 `场景 > 新场景` 或按 `Ctrl+N`
    *   将 **"Node3D"** 选择为根节点类型（用于 3D 游戏）或 **"Node2D"**（用于 2D 游戏）
    *   将根节点重命名为 `Player`
2.  **附加实体脚本** :
    
    *   选中根节点，点击"附加脚本"按钮（📄+图标）
    *   保存为 `e_player.gd`
3.  **保存场景** ：
    
    *   将文件保存为 `e_player.tscn` 到你的场景文件夹

**文件: `e_player.gd`**

```gdscript
# e_player.gd
class_name Player
extends Entity

func on_ready():
    # Sync the entity's scene position to the Transform component
    if has_component(C_Transform):
        var c_trs = get_component(C_Transform) as C_Transform
        c_trs.position = self.global_position
```

> 💡 **使用场景** : 玩家、敌人、投射物或任何需要在你的游戏世界中定位的对象。

### **选项 B：基于代码的实体** （纯数据容器）

当你不需要空间属性，只想使用一个纯数据容器时使用（例如，游戏管理器、抽象系统、计时器）。

```gdscript
# Just extend Entity directly
class_name GameManager
extends Entity

# No scene needed - instantiate with GameManager.new()
```

> 💡 **使用场景** ：游戏状态管理器、任务追踪器、背包系统或任何非空间的游戏逻辑。

* * *

**对于本教程** ，我们将使用 **选项 A**（基于场景的），因为我们希望玩家能在屏幕上移动，并具有位置信息。

## 📦 第3步：你的第一个组件（1分钟）

组件包含数据。让我们创建健康和变换组件：

**文件：`c_health.gd`**

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

**文件：`c_transform.gd`**

```gdscript
# c_transform.gd
class_name C_Transform
extends Component

@export var position: Vector3 = Vector3.ZERO

func _init(pos: Vector3 = Vector3.ZERO):
    position = pos
```

**文件: `c_velocity.gd`**

```gdscript
# c_velocity.gd
class_name C_Velocity
extends Component

@export var velocity: Vector3 = Vector3.ZERO

func _init(vel: Vector3 = Vector3.ZERO):
    velocity = vel
```

> 💡 **关键原则** : 组件仅持有数据，从不包含逻辑。将它们视为数据容器。⚠️ **重要提示** : 组件的 `_init` 函数要求所有参数必须有默认值，否则 Godot 将崩溃。

## ⚙️ 第 4 步：你的第一个系统（1 分钟）

系统包含操作具有特定组件的实体的逻辑。这个系统使实体在屏幕上移动:

**文件: `s_movement.gd`**

```gdscript
# s_movement.gd
class_name MovementSystem
extends System

func query():
    # Find all entities that have both transform and velocity
    return q.with_all([C_Transform, C_Velocity])

func process(entities: Array[Entity], components: Array, delta: float):
    # Process each entity in the array
    for entity in entities:
        var c_trs = entity.get_component(C_Transform) as C_Transform
        var c_velocity = entity.get_component(C_Velocity) as C_Velocity

        # Move the entity based on its velocity
        c_trs.position += c_velocity.velocity * delta

        # Update the actual entity position in the scene
        entity.global_position = c_trs.position

        # Bounce off screen edges (simple example)
        if c_trs.position.x > 10 or c_trs.position.x < -10:
            c_velocity.velocity.x *= -1
```

> 💡 **系统逻辑** : 查询找到具有所需组件的实体，process() 在每帧对每个实体运行移动逻辑。

## 🎬 第5步: 观看效果 (1分钟)

现在让我们将所有内容整合到主场景中:

### 创建主场景

1.  **创建一个新场景** ，以 `Node` 作为根节点
2.  **添加一个 World 节点** 作为子节点（添加子节点 > 搜索 "World"）
3.  **将此脚本**附加到根节点：

**文件: `main.gd`**

```gdscript
# main.gd
extends Node

@onready var world: World = $World

func _ready():
    ECS.world = world

    # Load and instantiate the player entity scene
    var player_scene = preload("res://e_player.tscn")  # Adjust path as needed
    var e_player = player_scene.instantiate() as Player

    # Add components to the entity
    e_player.add_components([
        C_Health.new(100),
        C_Transform.new(),
        C_Velocity.new(Vector3(2, 0, 0))  # Move right at 2 units/second
    ])

    add_child(e_player)  # Add to scene tree
    ECS.world.add_entity(e_player)  # Add to ECS world

    # Create the movement system
    var movement_system = MovementSystem.new()
    ECS.world.add_system(movement_system)

func _process(delta):
    # Process all systems
    if ECS.world:
        ECS.process(delta)
```

**运行你的项目！** 🎉 现在你已经设置了一个可运行的 ECS，其中玩家实体在屏幕上移动并在边缘反弹！MovementSystem 根据实体的速度组件更新实体位置。

> 💡 **基于场景的实体** : 注意我们加载并实例化了 `e_player.tscn` 场景，而不是调用 `Player.new()`。这是必要的，因为我们需要访问空间属性（位置）。对于不需要空间属性的实体，`Entity.new()` 工作得很好。

## 🎯 你刚刚构建的

恭喜！您已成功创建您的第一个 ECS 项目：

*   **实体** : Player - 用于存放组件的容器
*   **组件** : C\_Health, C\_Transform, C\_Velocity - 纯粹的数据容器
*   **系统** : MovementSystem - 基于速度移动实体的逻辑
*   **世界** : 管理实体和系统的容器

## 📈 下一步

现在你已经掌握了基础知识，这里是如何提升技能：

### 1\. 创建实体预制体（推荐）

不要在代码中创建实体，而是使用 Godot 的场景系统：

1.  **创建一个新场景** ，将你的 Entity 类作为根节点
2.  **添加视觉子节点** （MeshInstance3D、Sprite3D 等）
3.  **通过 define\_components()添加组件**或在 Inspector 中的 component\_resources 数组
4.  **保存为 .tscn 文件** （例如，`e_player.tscn`）
5.  **在主场景中加载并实例化**

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
        var c_trs = get_component(C_Transform) as C_Transform
        c_trs.position = self.global_position
```

### 2\. 组织你的主场景

使用经过验证的场景模式来构建你的主场景：

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

**优点：**

*   **视觉组织**在 Godot 编辑器中
*   **轻松的系统重排序**在组之间
*   **可复用的系统配置**

### 3\. 了解更多模式

### 🧠 理解概念

**→ [核心概念指南](CORE_CONCEPTS-zh-CN-translation.md)** \- 深入了解实体、组件、系统和关系

### 🔧 添加更多功能

尝试将这些添加到你的移动玩家：

*   **输入系统** \- 添加 C\_Input 组件和系统，使用箭头键控制移动
*   **多个实体** \- 创建更多具有不同速度的移动对象
*   **碰撞系统** \- 添加 C\_Collision 组件，检测实体相互碰撞的情况
*   **重力系统** \- 添加向下的速度，使实体下落

### 📚 学习最佳实践

**→ [最佳实践指南](BEST_PRACTICES-zh-CN-translation.md)** \- 编写可维护的 ECS 代码

### 🔧 探索高级功能

*   **[组件查询](COMPONENT_QUERIES-zh-CN-translation.md)** \- 按组件属性值进行筛选
*   **[关系](RELATIONSHIPS-zh-CN-translation.md)** \- 连接实体以实现复杂交互
*   **[观察者](OBSERVERS-zh-CN-translation.md)** \- 响应变化的反应系统
*   **[性能优化](PERFORMANCE_OPTIMIZATION-zh-CN-translation.md)** \- 让你的游戏运行快速

## ❓ 遇到问题？

### 玩家无响应？

*   确认在 `_process()` 中调用了 `ECS.process(delta)`
*   验证组件是否通过 `define_components()` 或 Inspector 添加到实体
*   确保系统已添加到世界
*   确保在实体的 `on_ready() 中调用 transform 同步`

### 无法访问位置/旋转属性？

*   ⚠️ **实体继承自 Node，而非 Node3D**：要访问空间属性，请以 `Node3D`（3D）或 `Node2D`（2D）作为根节点类型创建场景
*   将您的实体脚本（继承自 `Entity`）附加到 Node3D/Node2D 根节点
*   加载并实例化场景文件（不要对空间实体使用 `.new()`）
*   **如果您不需要空间属性** ：使用 `Entity.new()` 对纯数据容器来说完全没问题
*   有关这两种实体创建方法，请参考步骤2

### 控制台出现错误？

*   检查所有类都扩展了正确的基类
*   验证文件名与类名匹配
*   确保 GECS 插件已启用

**仍然卡住？** → [故障排除指南](TROUBLESHOOTING-zh-CN-translation.md)

## 🏆 接下来是什么？

现在您已经准备好使用 GECS 构建令人惊叹的游戏了！实体-组件-系统模式将帮助您：

*   **扩展您的游戏** \- 添加功能而不破坏现有代码
*   **重用代码** \- 组件和系统适用于不同的实体类型
*   **更容易调试** \- 数据和逻辑清晰分离
*   **优化性能** \- GECS 为您处理高效的查询

**准备好深入探索了吗？** 从 [核心概念](CORE_CONCEPTS-zh-CN-translation.md) 开始，真正理解是什么让 ECS 如此强大。

**需要帮助吗？** [加入我们的 Discord 社区](https://discord.gg/eB43XU2tmn)获取支持和讨论。

* * *

*学习 ECS 最好的方法就是用它来构建。从简单开始，随着你对模式的理解逐渐增加复杂性。*