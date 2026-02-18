# GECS 序列化

GECS 框架使用 Godot 的原生资源格式提供了一种强大的序列化系统，支持持久化游戏状态、存档系统和关卡数据管理。

## 快速入门

### 基本存档/加载

```gdscript
# Save entities with persistent components
var query = ECS.world.query.with_all([C_Persistent])
var data = ECS.serialize(query)
ECS.save(data, "user://savegame.tres")

# Load entities back
var entities = ECS.deserialize("user://savegame.tres")
for entity in entities:
    ECS.world.add_entity(entity)
```

### 二进制格式

```gdscript
# Save as binary for production (smaller files)
ECS.save(data, "user://savegame.tres", true)  # Creates .res file

# Load auto-detects format (tries .res first, then .tres)
var entities = ECS.deserialize("user://savegame.tres")
```

## API 参考

### ECS.serialize(query: QueryBuilder) -> GecsData

将符合查询条件的实体转换为可序列化数据。

**示例：**

```gdscript
# Serialize specific entities
var player_query = ECS.world.query.with_all([C_Player, C_Health])
var save_data = ECS.serialize(player_query)
```

### ECS.save(data: GecsData, filepath: String, binary: bool = false) -> bool

将数据保存到磁盘。成功时返回 `true`。

**参数：**

*   `data`: 序列化实体数据
*   `filepath`: 保存位置（使用 `.tres` 扩展名）
*   `binary`: 如果为 `true`，则保存为 `.res`（更小，加载更快）

### ECS.deserialize(filepath: String) -> Array\[Entity\]

从文件加载实体。如果文件不存在，则返回空数组。

**自动检测：** 首先尝试二进制 `.res`，如果失败则回退到文本 `.tres`。

## 组件序列化

仅序列化 `@export` 变量：

```gdscript
class_name C_PlayerData
extends Component

@export var health: float = 100.0        # ✅ Saved
@export var inventory: Array[String]     # ✅ Saved
@export var position: Vector2            # ✅ Saved

var _cache: Dictionary = {}              # ❌ Not saved
```

**支持类型：** 所有 Godot 内置类型（int, float, String, Vector2/3, Color, Array, Dictionary 等）。

## 用例

### 存档系统

```gdscript
func save_game(slot: String):
    var query = ECS.world.query.with_all([C_Persistent])
    var data = ECS.serialize(query)

    if ECS.save(data, "user://saves/slot_%s.tres" % slot, true):
        print("Game saved!")

func load_game(slot: String):
    ECS.world.purge()  # Clear current state

    var entities = ECS.deserialize("user://saves/slot_%s.tres" % slot)
    for entity in entities:
        ECS.world.add_entity(entity)
```

### 关卡导出/导入

```gdscript
func export_level():
    var query = ECS.world.query.with_all([C_LevelObject])
    var data = ECS.serialize(query)
    ECS.save(data, "res://levels/level_01.tres")

func load_level(path: String):
    var entities = ECS.deserialize(path)
    ECS.world.add_entities(entities)
```

### 选择性序列化

```gdscript
# Save only player data
var player_query = ECS.world.query.with_all([C_Player])

# Save entities in specific area
var area_query = ECS.world.query.with_group("area_1")

# Save entities with specific components
var combat_query = ECS.world.query.with_all([C_Health, C_Weapon])
```

## 数据结构

系统使用两个主要资源类：

### GecsData

```gdscript
class_name GecsData
extends Resource

@export var version: String = "0.1"
@export var entities: Array[GecsEntityData] = []
```

### GecsEntityData

```gdscript
class_name GecsEntityData
extends Resource

@export var entity_name: String = ""
@export var scene_path: String = ""      # For prefab entities
@export var components: Array[Component] = []
```

## 错误处理

```gdscript
# Serialize never fails (returns empty data if no matches)
var data = ECS.serialize(query)

# Check save success
if not ECS.save(data, filepath):
    print("Save failed - check permissions")

# Handle missing files
var entities = ECS.deserialize(filepath)
if entities.is_empty():
    print("No data loaded")
```

## 性能

*   **内存：** 序列化时创建组件副本
*   **速度：** 二进制格式约小 60%，加载比文本快
*   **比例：** 测试了 100 多个实体，亚秒级性能

## 二进制格式与文本格式

**文本 (.tres):**

*   人类可读
*   编辑器可检查
*   版本控制友好
*   开发调试

**二进制 (.res):**

*   更小的文件大小
*   更快的加载速度
*   生产版本构建
*   加载时自动检测

## 文件结构示例

```tres
[gd_resource type="GecsData" format=3]

[sub_resource type="C_Health" id="1"]
current = 85.0
maximum = 100.0

[sub_resource type="GecsEntityData" id="2"]
entity_name = "Player"
components = [SubResource("1")]

[resource]
version = "0.1"
entities = [SubResource("2")]
```

## 最佳实践

1.  **使用有意义的文件名：** `player_save.tres`, `level_boss.tres`
2.  **按用途组织：** `user://saves/`, `res://levels/`
3.  **优雅地处理缺失组件**
4.  **为生产环境使用二进制格式**
5.  **为兼容性版本化存档数据**
6.  **使用空查询结果进行测试**

## 限制

*   无实体关系（计划功能）
*   预制实体需要场景文件
*   外部资源引用需要手动处理