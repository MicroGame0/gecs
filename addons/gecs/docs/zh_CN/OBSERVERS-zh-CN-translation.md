# GECS 中的观察者

> **响应组件变化的反应系统**

观察者提供一种反应式编程模型，其中系统会自动响应组件的变化、添加和移除。这允许实现解耦的、事件驱动的游戏逻辑。

## 📋 前置条件

*   理解[核心概念](CORE_CONCEPTS-zh-CN-translation.md)
*   熟悉[系统](CORE_CONCEPTS-zh-CN-translation.md#systems)
*   观察者必须添加到世界中才能发挥作用

## 🎯 什么是观察者？

观察者是专门用于监视特定组件变化并立即做出反应的系统。它们不是每帧都处理实体，而是在实际发生变化时才触发。

**优点：**

*   **性能** \- 仅在发生变化时运行，而非每帧都运行
*   **解耦** \- 组件无需知道依赖它们的系统
*   **响应性** \- 对状态变化的即时响应
*   **清晰逻辑** \- 将变更处理逻辑与常规处理分离

## 🔧 观察者结构

观察者扩展了 `Observer` 类并实现了关键方法：

1.  **`watch()`** - 指定要监控的事件组件（ **必需** \- 如果未重写将导致崩溃）
2.  **`match()`** - 定义一个查询来过滤触发事件的实体（可选 - 默认为所有实体）
3.  **事件处理器** \- 处理特定类型的变更

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

## 🎮 观察事件类型

### on\_component\_added()

当被观察的组件被添加到实体时触发：

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

当被观察组件的属性发生变化时触发：

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

当被观察组件从实体中移除时触发：

```gdscript
class_name HealthUIObserver
extends Observer

func watch() -> Resource:
    return C_Health

func on_component_removed(entity: Entity, component: Resource):
    # Clean up health bar when health component is removed
    call_deferred("remove_health_bar", entity)
```

## 💡 常见的观察者模式

### 转换同步

保持实体场景变换与变换组件同步：

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

### 状态效果视觉效果

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

### 音频反馈

触发组件变化时的音效：

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

## 🏗️ 观察者最佳实践

### 命名规范

**观察者文件和类：**

*   **类名** : `DescriptiveNameObserver` (TransformObserver, HealthUIObserver)
*   **文件名** : `o_descriptive_name.gd` (o\_transform.gd, o\_health\_ui.gd)

### 使用延迟调用

始终使用 `call_deferred()` 来延迟执行工作，避免在组件更新时立即执行：

```gdscript
# ✅ Good - Defer work for later execution
func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    call_deferred("update_ui_element", entity, new_value)

# ❌ Avoid - Immediate execution can cause issues
func on_component_changed(entity: Entity, component: Resource, property: String, new_value: Variant, old_value: Variant):
    update_ui_element(entity, new_value)  # May cause timing issues
```

### 保持观察者逻辑简单

让观察者专注于单一职责：

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

### 使用具体查询

使用 `match()` 过滤触发观察者的实体：

```gdscript
# ✅ Good - Specific query
func match():
    return q.with_all([C_Health]).with_group("player")  # Only player health

# ❌ Avoid - Too broad
func match():
    return q.with_all([C_Health])  # ALL entities with health
```

## 🎯 何时使用观察者

**观察者用于：**

*   基于游戏状态变化的 UI 更新
*   由状态变化触发的音频/视觉效果
*   对关键状态变化（死亡、升级）的即时响应
*   组件与场景节点的同步
*   事件记录与分析

**使用常规系统进行：**

*   连续处理（移动、物理）
*   逐帧更新
*   依赖多个实体的复杂逻辑
*   性能关键的处理循环

## 🚀 向世界添加观察者

观察者必须注册到世界中才能发挥作用。有几种方法可以做到这一点：

### 手动注册

```gdscript
# In your scene or main script
func _ready():
    var health_observer = HealthUIObserver.new()
    ECS.world.add_observer(health_observer)
    
    # Or add multiple observers at once
    ECS.world.add_observers([health_observer, transform_observer, audio_observer])
```

### 自动场景树注册

在你的场景中将观察者节点放置在系统根目录下（默认："Systems"节点），它们将自动注册：

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

### 重要提示：

*   观察者节点会使用自己的 QueryBuilder 初始化（`observer.q`）
*   在注册过程中会调用 `watch()` 方法来验证组件
*   观察者必须从\``watch()`\`返回一个有效的 Component 类，否则会崩溃

## ⚠️ 常见问题与故障排除

### 观察者未触发

**问题** ：观察者事件从未触发 **解决方案** :

*   确保通过 `add_observer() 将观察者添加到世界`
*   检查 `watch()` 是否返回正确的组件类
*   验证实体是否匹配 `match()` 查询（如果定义）
*   组件变更必须在属性上，而不仅仅是内部状态

### 崩溃："你必须重写 watch()方法"

**问题** ：观察者在注册时崩溃 **解决方案** ：重写 `watch()` 方法并返回一个 Component 类：

```gdscript
func watch() -> Resource:
    return C_Health  # Must return actual component class
```

### 事件触发于错误实体

**问题** ：观察者触发了你不想的实体 **解决方案** ：使用 `match()` 来过滤实体：

```gdscript
func match():
    return q.with_all([C_Health]).with_group("player")  # Only players
```

### 属性变更未被检测到

**问题** : 观察者无法检测到组件属性的变化 **原因** :

*   直接对属性赋值应该自动生效
*   内部对象修改（如 Array.append()）可能不会触发信号
*   需要手动发送信号以进行复杂属性变更

## 📚 相关文档

*   **[核心概念](CORE_CONCEPTS-zh-CN-translation.md)** \- 理解 ECS 基础
*   **[系统](CORE_CONCEPTS-zh-CN-translation.md#systems)** \- 常规处理系统
*   **[最佳实践](BEST_PRACTICES-zh-CN-translation.md)** \- 编写可维护的 ECS 代码

* * *

*"观察者模式将您的 ECS 从轮询系统转变为响应式系统，使您的游戏能够智能地响应状态变化，而不是不断检查这些变化。"*