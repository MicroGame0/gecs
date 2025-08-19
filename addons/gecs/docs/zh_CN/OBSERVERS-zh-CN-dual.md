# Observers in GECS
GECS 中的观察者

> **Reactive systems that respond to component changes
> 响应组件变化的反应系统**

Observers provide a reactive programming model where systems automatically respond to component changes, additions, and removals. This allows for decoupled, event-driven game logic.
观察者提供一种反应式编程模型，其中系统会自动响应组件的变化、添加和移除。这允许实现解耦的、事件驱动的游戏逻辑。

## 📋 Prerequisites
📋 前置条件

*   Understanding of [Core Concepts](CORE_CONCEPTS.md)
    对核心概念的理解
*   Familiarity with [Systems](CORE_CONCEPTS.md#systems)
    系统熟悉度
*   Observers must be added to the World to function
    观察者必须添加到世界中才能发挥作用

## 🎯 What are Observers?
🎯 什么是观察者？

Observers are specialized systems that watch for changes to specific components and react immediately when those changes occur. Instead of processing entities every frame, observers only trigger when something actually changes.
观察者是专门用于监视特定组件变化并立即对这些变化做出反应的系统。观察者不会每帧处理实体，只有在实际发生变化时才会触发。

**Benefits:
优点：**

*   **Performance** - Only runs when changes occur, not every frame
    性能 - 仅在发生变化时运行，而非每帧都运行
*   **Decoupling** - Components don't need to know what systems depend on them
    解耦 - 组件无需知道依赖它们的系统
*   **Reactivity** - Immediate response to state changes
    响应性 - 对状态变化立即响应
*   **Clean Logic** - Separate change-handling logic from regular processing
    清晰的逻辑 - 将变更处理逻辑与常规处理分离

## 🔧 Observer Structure
🔧 观察者结构

Observers extend the `Observer` class and implement key methods:
观察者扩展 `Observer` 类并实现关键方法：

1.  **`watch()`** - Specifies which component to monitor for events (**required** - will crash if not overridden)
    `watch()` - 指定要监控事件哪个组件（必需 - 如果未重写将崩溃）
2.  **`match()`** - Defines a query to filter which entities trigger events (optional - defaults to all entities)
    `match()` - 定义查询以过滤触发事件的实体（可选 - 默认为所有实体）
3.  **Event Handlers** - Handle specific types of changes
    事件处理器 - 处理特定类型的变更

```gdscript
# o_transform.gd
class_name TransformObserver
extends Observer

func watch() -> Resource:
    return C_Transform  # Watch for transform component changes (REQUIRED)

func on_component_added(entity: Entity, component: Resource):
    # Sync component transform to entity when added
    var transform_comp = component as C_Transform
    entity.global_transform = transform_comp.transform

func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    # Sync component transform to entity when changed
    var transform_comp = component as C_Transform
    entity.global_transform = transform_comp.transform
```

## 🎮 Observer Event Types
🎮 观察者事件类型

### on\_component\_added()

Triggered when a watched component is added to an entity:
当监视的组件被添加到实体时触发：

```gdscript
class_name HealthUIObserver
extends Observer

func watch() -> Resource:
    return C_Health

func match():
    return q.with_all([C_Health]).with_group("player")

func on_component_added(entity: Entity, component: Resource):
    # Create health bar when player gains health component
    var health = component as C_Health
    # Use call_deferred to avoid timing issues during component changes
    call_deferred("create_health_bar", entity, health.maximum)
```

### on\_component\_changed()

Triggered when a watched component's property changes:
当监视的组件的属性发生变化时触发：

```gdscript
class_name HealthBarObserver
extends Observer

func watch() -> Resource:
    return C_Health

func match():
    return q.with_all([C_Health]).with_group("player")

func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    if property == "current":
        var health = component as C_Health
        # Update health bar display
        call_deferred("update_health_bar", entity, health.current, health.maximum)
```

### on\_component\_removed()

Triggered when a watched component is removed from an entity:
当监视的组件从实体中移除时触发：

```gdscript
class_name HealthUIObserver
extends Observer

func watch() -> Resource:
    return C_Health

func on_component_removed(entity: Entity, component: Resource):
    # Clean up health bar when health component is removed
    call_deferred("remove_health_bar", entity)
```

## 💡 Common Observer Patterns
💡 常见观察者模式

### Transform Synchronization
转换同步

Keep entity scene transforms in sync with Transform components:
保持实体场景转换与转换组件同步

```gdscript
# o_transform.gd
class_name TransformObserver
extends Observer

func watch() -> Resource:
    return C_Transform

func on_component_added(entity: Entity, component: Resource):
    var transform_comp = component as C_Transform
    entity.global_transform = transform_comp.transform

func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    var transform_comp = component as C_Transform
    entity.global_transform = transform_comp.transform
```

### Status Effect Visuals
状态效果视觉效果

Show visual feedback for status effects:
为状态效果显示视觉反馈：

```gdscript
# o_status_effects.gd
class_name StatusEffectObserver
extends Observer

func watch() -> Resource:
    return C_StatusEffect

func on_component_added(entity: Entity, component: Resource):
    var status = component as C_StatusEffect
    call_deferred("add_status_visual", entity, status.effect_type)

func on_component_removed(entity: Entity, component: Resource):
    var status = component as C_StatusEffect
    call_deferred("remove_status_visual", entity, status.effect_type)

func add_status_visual(entity: Entity, effect_type: String):
    match effect_type:
        "poison":
            # Add poison particle effect
            pass
        "shield":
            # Add shield visual overlay
            pass

func remove_status_visual(entity: Entity, effect_type: String):
    # Remove corresponding visual effect
    pass
```

### Audio Feedback
音频反馈

Trigger sound effects on component changes:
在组件变化时触发音效：

```gdscript
# o_audio_feedback.gd
class_name AudioFeedbackObserver
extends Observer

func watch() -> Resource:
    return C_Health

func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    if property == "current":
        var health_change = new_value - old_value
        
        if health_change < 0:
            # Health decreased - play damage sound
            call_deferred("play_damage_sound", entity.global_position)
        elif health_change > 0:
            # Health increased - play heal sound
            call_deferred("play_heal_sound", entity.global_position)
```

## 🏗️ Observer Best Practices
🏗️ 观察者最佳实践

### Naming Conventions
命名规范

**Observer files and classes:
观察者文件和类：**

*   **Class names**: `DescriptiveNameObserver` (TransformObserver, HealthUIObserver)
    类名： `DescriptiveNameObserver` (TransformObserver, HealthUIObserver)
*   **File names**: `o_descriptive_name.gd` (o\_transform.gd, o\_health\_ui.gd)
    文件名： `o_descriptive_name.gd` (o\_transform.gd, o\_health\_ui.gd)

### Use Deferred Calls
使用延迟调用

Always use `call_deferred()` to defer work and avoid immediate execution during component updates:
始终使用 `call_deferred()` 来延迟工作，并在组件更新时避免立即执行：

```gdscript
# ✅ Good - Defer work for later execution
func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    call_deferred("update_ui_element", entity, new_value)

# ❌ Avoid - Immediate execution can cause issues
func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    update_ui_element(entity, new_value)  # May cause timing issues
```

### Keep Observer Logic Simple
保持观察者逻辑简单

Focus observers on single responsibilities:
聚焦观察者于单一职责：

```gdscript
# ✅ Good - Single purpose observer
class_name HealthUIObserver
extends Observer

func watch() -> Resource:
    return C_Health

func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    if property == "current":
        call_deferred("update_health_display", entity, new_value)

# ❌ Avoid - Observer doing too much
class_name HealthObserver
extends Observer

func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    # Too many responsibilities in one observer
    update_health_display(entity, new_value)
    play_damage_sound(entity)
    check_achievements(entity)
    save_game_state()
```

### Use Specific Queries
使用具体查询

Filter which entities trigger observers with `match()`:
过滤触发观察者的实体，使用 `match()` ：

```gdscript
# ✅ Good - Specific query
func match():
    return q.with_all([C_Health]).with_group("player")  # Only player health

# ❌ Avoid - Too broad
func match():
    return q.with_all([C_Health])  # ALL entities with health
```

## 🎯 When to Use Observers
🎯 使用观察者的时机

**Use Observers for:
使用观察者来：**

*   UI updates based on game state changes
    基于游戏状态变化的 UI 更新
*   Audio/visual effects triggered by state changes
    由状态变化触发的音频/视觉效果
*   Immediate response to critical state changes (death, level up)
    对关键状态变化的即时响应（死亡、升级）
*   Synchronization between components and scene nodes
    组件与场景节点的同步
*   Event logging and analytics
    事件日志记录与分析

**Use Regular Systems for:
使用常规系统进行：**

*   Continuous processing (movement, physics)
    持续处理（移动、物理）
*   Frame-by-frame updates
    逐帧更新
*   Complex logic that depends on multiple entities
    依赖多个实体的复杂逻辑
*   Performance-critical processing loops
    性能关键的处理循环

## 🚀 Adding Observers to the World
🚀 向世界添加观察者

Observers must be registered with the World to function. There are several ways to do this:
观察者必须向世界注册才能使用。有几种方法可以做到这一点：

### Manual Registration
手动注册

```gdscript
# In your scene or main script
func _ready():
    var health_observer = HealthUIObserver.new()
    ECS.world.add_observer(health_observer)
    
    # Or add multiple observers at once
    ECS.world.add_observers([health_observer, transform_observer, audio_observer])
```

### Automatic Scene Tree Registration
自动场景树注册

Place Observer nodes in your scene under the systems root (default: "Systems" node), and they'll be automatically registered:
将观察者节点放置在您的场景中系统根目录下（默认："Systems"节点），它们将自动注册：

```
Main
├── World
├── Systems/          # Observers placed here are auto-registered
│   ├── HealthUIObserver
│   ├── TransformObserver
│   └── AudioFeedbackObserver
└── Entities/
    └── Player
```

### Important Notes:
重要提示：

*   Observers are initialized with their own QueryBuilder (`observer.q`)
    观察者使用自己的 QueryBuilder（ `observer.q` ）初始化
*   The `watch()` method is called during registration to validate the component
    注册期间会调用 `watch()` 方法来验证组件
*   Observers must return a valid Component class from `watch()` or they'll crash
    观察者必须从 `watch()` 返回一个有效的 Component 类，否则会崩溃

## ⚠️ Common Issues & Troubleshooting
⚠️ 常见问题与故障排除

### Observer Not Triggering
观察者未触发

**Problem**: Observer events never fire **Solutions**:
问题：观察者事件从未触发 解决方案：

*   Ensure the observer is added to the World with `add_observer()`
    确保观察者已添加到世界（使用 `add_observer()` ）
*   Check that `watch()` returns the correct component class
    确认 `watch()` 返回正确的组件类
*   Verify entities match the `match()` query (if defined)
    验证实体是否匹配 `match()` 查询（如果已定义）
*   Component changes must be on properties, not just internal state
    组件变更必须在属性上，而不仅仅是内部状态

### Crash: "You must override the watch() method"
崩溃： "你必须重写 watch() 方法"

**Problem**: Observer crashes on registration **Solution**: Override `watch()` method and return a Component class:
问题：Observer 在注册时崩溃 解决方案：重写 `watch()` 方法并返回一个 Component 类：

```gdscript
func watch() -> Resource:
    return C_Health  # Must return actual component class
```

### Events Fire for Wrong Entities
事件为错误实体触发

**Problem**: Observer triggers for entities you don't want **Solution**: Use `match()` to filter entities:
问题：Observer 为你不想触发实体的触发 解决方案：使用 `match()` 来过滤实体：

```gdscript
func match():
    return q.with_all([C_Health]).with_group("player")  # Only players
```

### Property Changes Not Detected
属性变化未被检测到

**Problem**: Observer doesn't detect component property changes **Causes**:
问题：观察者无法检测到组件属性的变化 原因：

*   Direct assignment to properties should work automatically
    直接对属性赋值应该自动生效
*   Internal object modifications (like Array.append()) may not trigger signals
    内部对象修改（如 Array.append()）可能不会触发信号
*   Manual signal emission required for complex property changes
    复杂属性变化需要手动发射信号

## 📚 Related Documentation
📚 相关文档

*   **[Core Concepts](CORE_CONCEPTS.md)** - Understanding the ECS fundamentals
    核心概念 - 理解 ECS 基础
*   **[Systems](CORE_CONCEPTS.md#systems)** - Regular processing systems
    系统组件 - 常规处理系统
*   **[Best Practices](BEST_PRACTICES.md)** - Write maintainable ECS code
    最佳实践 - 编写可维护的 ECS 代码

* * *

*"Observers turn your ECS from a polling system into a reactive system, making your game respond intelligently to state changes rather than constantly checking for them."
"Observers 可以将您的 ECS 从轮询系统转变为响应式系统，使您的游戏能够智能地响应状态变化，而不是不断检查这些变化。"*