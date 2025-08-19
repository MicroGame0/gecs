# GECS Troubleshooting Guide
GECS 故障排除指南

> **Quickly solve common GECS issues
> 快速解决常见的 GECS 问题**

This guide helps you diagnose and fix the most common problems when working with GECS. Find your issue, apply the solution, and learn how to prevent it.
本指南帮助你诊断和修复在使用 GECS 时遇到的最常见问题。找到你的问题，应用解决方案，并学习如何预防它。

## 📋 Quick Diagnosis
📋 快速诊断

### My Game Isn't Working At All
我的游戏完全无法运行

**Symptoms**: No entities moving, systems not running, nothing happening
症状：没有实体移动，系统未运行，没有任何反应

**Quick Check**:
快速检查：

```gdscript
# In your _process() method, ensure you have:
func _process(delta):
    if ECS.world:
        ECS.world.process(delta)  # This line is critical!
```

**Missing this?** → [Systems Not Running](#systems-not-running)
缺少这个？→ 系统未运行

### Entities Aren't Moving/Updating
实体未移动/更新

**Symptoms**: Entities exist but don't respond to systems
症状：实体存在但系统无响应

**Quick Check**:
快速检查：

1.  Are your entities added to the world? `ECS.world.add_entity(entity)`
    你的实体是否已添加到世界？ `ECS.world.add_entity(entity)`
2.  Do your entities have the right components? Check system queries
    你的实体包含正确的组件吗？检查系统查询
3.  Are your systems properly organized in scene hierarchy? Check default\_systems.tscn
    你的系统在场景层级中是否正确组织？检查 default\_systems.tscn

**Still broken?** → [Entity Issues](#entity-issues)
仍然出问题？→ 实体问题

### Performance Is Terrible
性能太差

**Symptoms**: Low FPS, stuttering, slow response
症状：低 FPS、卡顿、响应缓慢

**Quick Check**:
快速检查：

1.  Enable profiling: `ECS.world.enable_profiling = true`
    启用性能分析： `ECS.world.enable_profiling = true`
2.  Check entity count: `print(ECS.world.entity_count)`
    检查实体数量： `print(ECS.world.entity_count)`
3.  Look for expensive queries in your systems
    寻找系统中的昂贵查询

**Need optimization?** → [Performance Issues](#performance-issues)
需要优化？→ 性能问题

## 🚫 Systems Not Running
🚫 系统未运行

### Problem: Systems Never Execute
问题：系统从未执行

**Error Messages**:
错误消息：

*   No error, but `process()` method never called
    没有错误，但 `process()` 方法从未被调用
*   Entities exist but don't change
    实体存在但未发生变化

**Solution**:
解决方案：

```gdscript
# ✅ Ensure this exists in your main scene
func _process(delta):
    ECS.process(delta)  # This processes all systems

# OR if using system groups:
func _process(delta):
    ECS.process(delta, "physics")
    ECS.process(delta, "render")
```

**Prevention**: Always call `ECS.process()` in your main game loop.
预防：始终在主游戏循环中调用 `ECS.process()` 。

### Problem: System Query Returns Empty
问题：系统查询返回空

**Symptoms**: System exists but `process()` never called
症状：系统存在但 `process()` 从未被调用

**Diagnosis**:
诊断：

```gdscript
# Add this to your system for debugging
class_name MySystem extends System

func _ready():
    print("MySystem query result: ", query().execute().size())

func query():
    return q.with_all([C_ComponentA, C_ComponentB])
```

**Common Causes**:
常见原因：

1.  **Missing Components**:
    缺失组件：
    
    ```gdscript
    # ❌ Problem - Entity missing required component
    var entity = Entity.new()
    entity.add_component(C_ComponentA.new())
    # Missing C_ComponentB!
    
    # ✅ Solution - Add all required components
    entity.add_component(C_ComponentA.new())
    entity.add_component(C_ComponentB.new())
    ```
    
2.  **Wrong Component Types**:
    组件类型错误：
    
    ```gdscript
    # ❌ Problem - Using instance instead of class
    func query():
        return q.with_all([C_ComponentA.new()])  # Wrong!
    
    # ✅ Solution - Use class reference
    func query():
        return q.with_all([C_ComponentA])  # Correct!
    ```
    
3.  **Component Not Added to World**:
    组件未添加到世界：
    
    ```gdscript
    # ❌ Problem - Entity not in world
    var entity = Entity.new()
    entity.add_component(C_ComponentA.new())
    # Entity never added to world!
    
    # ✅ Solution - Add entity to world
    ECS.world.add_entity(entity)
    ```
    

## 🎭 Entity Issues
🎭 实体问题

### Problem: Entity Components Not Found
问题：未找到实体组件

**Error Messages**:
错误信息：

*   `get_component() returned null`
*   `Entity does not have component of type...`

**Diagnosis**:
诊断：

```gdscript
# Debug what components an entity actually has
func debug_entity_components(entity: Entity):
    print("Entity components:")
    for component_path in entity.components.keys():
        print("  ", component_path)
```

**Solution**: Ensure components are added correctly:
解决方案：确保组件已正确添加：

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

### Problem: Component Properties Not Updating
问题：组件属性未更新

**Symptoms**: Setting component properties has no effect
症状：设置组件属性没有效果

**Common Causes**:
常见原因：

1.  **Getting Component Reference Once**:
    获取组件引用一次：
    
    ```gdscript
    # ❌ Problem - Stale component reference
    var health = entity.get_component(C_Health)
    # ... later in code, component gets replaced ...
    health.current = 50  # Updates old component!
    
    # ✅ Solution - Get fresh reference each time
    entity.get_component(C_Health).current = 50
    ```
    
2.  **Modifying Wrong Entity**:
    修改错误的实体：
    
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
    

## 💥 Common Errors
💥 常见错误

### Error: "Cannot access property/method on null instance"
错误："无法在空实例上访问属性/方法"

**Full Error**:
完整错误：

```
Invalid get index 'current' (on base: 'null instance')
```

**Cause**: Component doesn't exist on entity
原因：实体上不存在该组件

**Solution**:
解决方案：

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

### Error: "Class not found"
错误："找不到类"

**Full Error**:
完整错误：

```
Identifier 'ComponentName' not found in current scope
```

**Causes & Solutions**:
原因及解决方案：

1.  **Missing class\_name**:
    缺少 class\_name：
    
    ```gdscript
    # ❌ Problem - No class_name declaration
    extends Component
    # Script exists but can't be referenced by name
    
    # ✅ Solution - Add class_name
    class_name C_Health
    extends Component
    ```
    
2.  **File not saved or loaded**:
    文件未保存或加载：
    
    *   Save your component script files
        保存你的组件脚本文件
    *   Restart Godot if classes still not found
        如果类仍然找不到，重启 Godot
    *   Check for syntax errors in the component file
        检查组件文件中的语法错误
3.  **Wrong inheritance**:
    错误的继承:
    
    ```gdscript
    # ❌ Problem - Wrong base class
    class_name C_Health
    extends Node  # Should be Component!
    
    # ✅ Solution - Correct inheritance
    class_name C_Health
    extends Component
    ```
    

## 🐌 Performance Issues
🐌 性能问题

### Problem: Low FPS / Stuttering
问题：低 FPS / 卡顿

**Diagnosis Steps**:
诊断步骤：

1.  **Enable profiling**:
    启用性能分析：
    
    ```gdscript
    ECS.world.enable_profiling = true
    
    # Check processing times
    func _process(delta):
        ECS.process(delta)
        print("Frame time: ", get_process_delta_time() * 1000, "ms")
    ```
    
2.  **Check entity count**:
    检查实体数量：
    
    ```gdscript
    print("Total entities: ", ECS.world.entity_count)
    print("System count: ", ECS.world.get_system_count())
    ```
    

**Common Fixes**:
常见修复：

1.  **Too Many Entities in Broad Queries**:
    宽泛查询中实体过多：
    
    ```gdscript
    # ❌ Problem - Overly broad query
    func query():
        return q.with_all([C_Position])  # Matches everything!
    
    # ✅ Solution - More specific query
    func query():
        return q.with_all([C_Position, C_Movable])
    ```
    
2.  **Expensive Queries Rebuilt Every Frame**:
    每帧重建昂贵查询：
    
    ```gdscript
    # ❌ Problem - Rebuilding queries
    func process_all(delta: float):
        var entities = ECS.world.query.with_all([C_ComponentA]).execute()
    
    # ✅ Solution - Cache queries
    var _cached_query: Query
    func _ready():
        _cached_query = ECS.world.query.with_all([C_ComponentA]).build()
    
    func process_all(delta: float):
        var entities = _cached_query.execute()
    ```
    

## 🔧 Integration Issues
🔧 集成问题

### Problem: GECS Conflicts with Godot Features
问题：GECS 与 Godot 功能冲突

**Issue**: Using GECS entities with Godot nodes causes problems
问题：使用 GECS 实体与 Godot 节点会导致问题

**Solution**: Choose your approach consistently:
解决方案：始终一致地选择你的方法：

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

**Avoid**: Mixing approaches inconsistently in the same project.
避免：在同一项目中不一致地混合方法。

### Problem: GECS Not Working After Scene Changes
问题：场景更改后 GECS 无法工作

**Symptoms**: Systems stop working when changing scenes
症状：更改场景时系统停止工作

**Solution**: Properly reinitialize ECS in new scenes:
解决方案：在新场景中正确重新初始化 ECS：

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

**Prevention**: Always initialize ECS properly in each scene that uses it.
预防：在所有使用它的场景中，始终正确初始化 ECS。

## 🛠️ Debugging Tools
🛠️ 调试工具

### Enable Debug Logging
启用调试日志

Add to your project settings or main script:
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

### Entity Inspector Tool
实体检查工具

Create a debug tool to inspect entities at runtime:
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

## 📚 Getting More Help
📚 获取更多帮助

### Community Resources
社区资源

*   **Discord**: [Join our community](https://discord.gg/eB43XU2tmn) for help and discussions
    Discord: 加入我们的社区以获得帮助和参与讨论
*   **GitHub Issues**: [Report bugs](https://github.com/csprance/gecs/issues)
    GitHub Issues: 报告错误
*   **Documentation**: [Complete Guide](../DOCUMENTATION-zh-CN-dual.md)
    文档: 完整指南

### Before Asking for Help
在寻求帮助前

Include this information in your question:
在你的问题中包含以下信息：

1.  **GECS version** you're using
    你正在使用的 GECS 版本
2.  **Godot version** you're using
    你正在使用的 Godot 版本
3.  **Minimal code example** that reproduces the issue
    一个能够重现问题的最小代码示例
4.  **Error messages** (full text, not paraphrased)
    错误消息（完整文本，非改写）
5.  **Expected vs actual behavior
    预期行为与实际行为**

### Still Stuck?
仍然卡住？

If this guide doesn't solve your problem:
如果本指南无法解决您的问题：

1.  **Check the examples** in [Getting Started](GETTING_STARTED-zh-CN-dual.md)
    检查入门指南中的示例
2.  **Review best practices** in [Best Practices](BEST_PRACTICES-zh-CN-dual.md)
    在最佳实践中查看最佳实践
3.  **Search GitHub issues** for similar problems
    在 GitHub 问题中搜索类似问题
4.  **Create a minimal reproduction** and ask for help
    创建最小化复现并寻求帮助

* * *

*"Every bug is a learning opportunity. The key is knowing where to look and what questions to ask."
"每个错误都是学习的机会。关键在于知道去哪里寻找以及该问什么问题。"*
