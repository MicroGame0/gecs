# GECS 故障排除指南

> **快速解决常见的 GECS 问题**

本指南帮助你诊断和修复在使用 GECS 时遇到的最常见问题。找到你的问题，应用解决方案，并学习如何预防它。

## 📋 快速诊断

### 我的游戏完全无法运行

**症状** ：没有实体移动，系统未运行，没有任何反应

**快速检查** :

```gdscript
# In your _process() method, ensure you have:
func _process(delta):
    if ECS.world:
        ECS.world.process(delta)  # This line is critical!
```

**缺少这个？** → [系统未运行](#systems-not-running)

### 实体未移动/更新

**症状** ：实体存在但系统无响应

**快速检查** ：

1.  你的实体是否已添加到世界？`ECS.world.add_entity(entity)`
2.  您的实体是否包含正确的组件？检查系统查询
3.  您的系统是否在场景层级中正确组织？检查 default\_systems.tscn

**仍然出问题？** → [实体问题](#entity-issues)

### 性能太差

**症状** : 低 FPS、卡顿、响应缓慢

**快速检查** :

1.  启用性能分析： `ECS.world.enable_profiling = true`
2.  检查实体数量：`print(ECS.world.entity_count)`
3.  查找系统中的昂贵查询

**需要优化？** → [性能问题](#performance-issues)

## 🚫 系统未运行

### 问题：系统从未执行

**错误信息** :

*   没有错误，但 `process()` 方法从未被调用
*   实体存在但未发生变化

**解决方案** :

```gdscript
# ✅ Ensure this exists in your main scene
func _process(delta):
    ECS.process(delta)  # This processes all systems

# OR if using system groups:
func _process(delta):
    ECS.process(delta, "physics")
    ECS.process(delta, "render")
```

**预防** : 始终在主游戏循环中调用 `ECS.process()`。

### 问题：系统查询返回空

**症状** ：系统存在但 `process()` 从未被调用

**诊断** ：

```gdscript
# Add this to your system for debugging
class_name MySystem extends System

func _ready():
    print("MySystem query result: ", query().execute().size())

func query():
    return q.with_all([C_ComponentA, C_ComponentB])
```

**常见原因** :

1.  **缺少组件** :
    
    ```gdscript
    # ❌ Problem - Entity missing required component
    var entity = Entity.new()
    entity.add_component(C_ComponentA.new())
    # Missing C_ComponentB!
    
    # ✅ Solution - Add all required components
    entity.add_component(C_ComponentA.new())
    entity.add_component(C_ComponentB.new())
    ```
    
2.  **组件类型错误** :
    
    ```gdscript
    # ❌ Problem - Using instance instead of class
    func query():
        return q.with_all([C_ComponentA.new()])  # Wrong!
    
    # ✅ Solution - Use class reference
    func query():
        return q.with_all([C_ComponentA])  # Correct!
    ```
    
3.  **组件未添加到世界** :
    
    ```gdscript
    # ❌ Problem - Entity not in world
    var entity = Entity.new()
    entity.add_component(C_ComponentA.new())
    # Entity never added to world!
    
    # ✅ Solution - Add entity to world
    ECS.world.add_entity(entity)
    ```
    

## 🎭 实体问题

### 问题：未找到实体组件

**错误信息** :

*   `get_component() returned null`
*   `Entity does not have component of type...`

**诊断** :

```gdscript
# Debug what components an entity actually has
func debug_entity_components(entity: Entity):
    print("Entity components:")
    for component_path in entity.components.keys():
        print("  ", component_path)
```

**解决方案** : 确保组件已正确添加：

```gdscript
# ✅ Correct component addition
var entity = Entity.new()
entity.add_component(C_Health.new(100))
entity.add_component(C_Position.new(Vector2(50, 50)))

# Verify component exists before using
if entity.has_component(C_Health):
    var health = entity.get_component(C_Health)
    health.current -= 10
```

### <strong>问题 </strong>: 组件属性未更新

**症状** : 设置组件属性无效

**常见原因** :

1.  **一次获取组件引用** :
    
    ```gdscript
    # ❌ Problem - Stale component reference
    var health = entity.get_component(C_Health)
    # ... later in code, component gets replaced ...
    health.current = 50  # Updates old component!
    
    # ✅ Solution - Get fresh reference each time
    entity.get_component(C_Health).current = 50
    ```
    
2.  **修改错误的实体** :
    
    ```gdscript
    # ❌ Problem - Variable confusion
    var player = get_player_entity()
    var enemy = get_enemy_entity()
    
    # Accidentally modify wrong entity
    player.get_component(C_Health).current = 0  # Meant to be enemy!
    
    # ✅ Solution - Use clear variable names
    var player_health = player.get_component(C_Health)
    var enemy_health = enemy.get_component(C_Health)
    enemy_health.current = 0
    ```
    

## 💥 常见错误

### 错误："无法在空实例上访问属性/方法"

**完整错误** :

```
Invalid get index 'current' (on base: 'null instance')
```

**原因** : 实体上不存在该组件

**解决方案** :

```gdscript
# ❌ Causes null error
var health = entity.get_component(C_Health)
health.current -= 10  # health is null!

# ✅ Safe component access
if entity.has_component(C_Health):
    var health = entity.get_component(C_Health)
    health.current -= 10
else:
    print("Entity doesn't have C_Health!")
```

### 错误: "找不到类"

**完整错误** :

```
Identifier 'ComponentName' not found in current scope
```

**原因及解决方案** :

1.  **缺少 class\_name**:
    
    ```gdscript
    # ❌ Problem - No class_name declaration
    extends Component
    # Script exists but can't be referenced by name
    
    # ✅ Solution - Add class_name
    class_name C_Health
    extends Component
    ```
    
2.  **文件未保存或加载** :
    
    *   保存你的组件脚本文件
    *   如果类仍然找不到，重启 Godot
    *   检查组件文件中的语法错误
3.  **继承错误** ：
    
    ```gdscript
    # ❌ Problem - Wrong base class
    class_name C_Health
    extends Node  # Should be Component!
    
    # ✅ Solution - Correct inheritance
    class_name C_Health
    extends Component
    ```
    

## 🐌 性能问题

### 问题：低 FPS / 卡顿

**诊断步骤** :

1.  **启用性能分析** :
    
    ```gdscript
    ECS.world.enable_profiling = true
    
    # Check processing times
    func _process(delta):
        ECS.process(delta)
        print("Frame time: ", get_process_delta_time() * 1000, "ms")
    ```
    
2.  **检查实体数量** :
    
    ```gdscript
    print("Total entities: ", ECS.world.entity_count)
    print("System count: ", ECS.world.get_system_count())
    ```
    

**常见修复方法** :

1.  **宽泛查询中实体过多** :
    
    ```gdscript
    # ❌ Problem - Overly broad query
    func query():
        return q.with_all([C_Position])  # Matches everything!
    
    # ✅ Solution - More specific query
    func query():
        return q.with_all([C_Position, C_Movable])
    ```
    
2.  **每帧重建昂贵查询** :
    
    ```gdscript
    # ❌ Problem - Rebuilding queries in process
    func process(entities: Array[Entity], components: Array, delta: float):
        var custom_entities = ECS.world.query.with_all([C_ComponentA]).execute()
    
    # ✅ Solution - Use the system's query() method (automatically cached)
    func query():
        return q.with_all([C_ComponentA])  # Automatically cached by GECS
    
    func process(entities: Array[Entity], components: Array, delta: float):
        # Just process the entities passed in - already filtered by query
        for entity in entities:
            # Process entity...
    ```
    

## 🔧 集成问题

### 问题：GECS 与 Godot 功能冲突

**问题** ：使用 GECS 实体与 Godot 节点会导致问题

**解决方案** ：始终一致地选择你的方法：

```gdscript
# ✅ Approach 1 - Pure ECS (recommended for complex games)
# Entities are not nodes, use ECS for everything
var entity = Entity.new()  # Not added to scene tree
entity.add_component(C_Position.new())
ECS.world.add_entity(entity)

# ✅ Approach 2 - Hybrid (good for simpler games)
# Entities are nodes, use ECS for specific systems
var entity = Entity.new()
add_child(entity)  # Entity is in scene tree
entity.add_component(C_Health.new())
ECS.world.add_entity(entity)
```

**避免** : 在同一个项目中不一致地混合方法。

### 问题：场景更改后 GECS 无法工作

**症状** ：更改场景时系统停止工作

**解决方案** ：在新场景中正确重新初始化 ECS：

```gdscript
# In each main scene script
func _ready():
    # Create new world for this scene
    var world = World.new()
    add_child(world)
    ECS.world = world

    # Systems are usually managed via scene composition
    # See default_systems.tscn for organization

    # Create your entities
    setup_entities()
```

**预防** ：在所有使用 ECS 的场景中，始终正确初始化 ECS。

## 🛠️ 调试工具

### 启用调试日志记录

添加到你的项目设置或主脚本中：

```gdscript
# Enable GECS debug output
ECS.set_debug_level(ECS.DEBUG_VERBOSE)

# This will show:
# - Entity creation/destruction
# - Component additions/removals
# - System processing information
# - Query execution details
```

### 实体检查工具

创建一个用于运行时检查实体的调试工具：

```gdscript
# DebugPanel.gd
extends Control

func _on_inspect_button_pressed():
    var entities = ECS.world.get_all_entities()
    print("=== ENTITY INSPECTOR ===")

    for i in range(min(10, entities.size())):  # Show first 10
        var entity = entities[i]
        print("Entity ", i, ":")
        print("  Components: ", entity.components.keys())
        print("  Groups: ", entity.get_groups())

        # Show component values
        for comp_path in entity.components.keys():
            var comp = entity.components[comp_path]
            print("    ", comp_path, ": ", comp)
```

## 📚 获取更多帮助

### 社区资源

*   **Discord**: [加入我们的社区](https://discord.gg/eB43XU2tmn) 获取帮助和参与讨论
*   **GitHub Issues**: [报告错误](https://github.com/csprance/gecs/issues)
*   **文档** : [完整指南](../DOCUMENTATION-zh-CN-translation.md)

### 在请求帮助前

在问题中包含以下信息：

1.  **你正在使用的 GECS 版本**
2.  **你正在使用的 Godot 版本**
3.  **能够重现问题的最小代码示例**
4.  **错误消息** （完整文本，非释义）
5.  **预期行为与实际行为**

### 仍然卡住？

如果本指南无法解决您的问题：

1.  **查看示例** 在 [入门指南](GETTING_STARTED-zh-CN-translation.md)
2.  **查阅最佳实践** 在 [最佳实践](BEST_PRACTICES-zh-CN-translation.md)
3.  **在 GitHub 问题中搜索** 相似问题
4.  **创建最小化复现案例** 并寻求帮助

* * *

*"每个错误都是学习的机会。关键在于知道去哪里寻找以及该问什么问题。"*