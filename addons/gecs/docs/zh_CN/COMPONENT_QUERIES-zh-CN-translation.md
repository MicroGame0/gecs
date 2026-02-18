# GECS 中的组件查询

> **高级基于属性的实体过滤**

组件查询提供了一种强大的方式来过滤实体，不仅基于组件的存在，还基于组件中的数据。这使得在游戏系统中能够进行精确的、数据驱动的实体选择。

## 📋 前置条件

*   理解[核心概念](CORE_CONCEPTS-zh-CN-translation.md)
*   熟悉[基本查询](CORE_CONCEPTS-zh-CN-translation.md#query-system)

## 🎯 简介

在标准的 ECS 查询中，您通过实体拥有或不拥有的组件来过滤实体。组件查询在此基础上更进一步，允许您根据组件内的**值**进行过滤。

您不仅可以询问"哪些实体拥有 HealthComponent？"，还可以询问"哪些实体的 HealthComponent 当前健康值小于 20？"

## 使用组件查询与 `QueryBuilder`

`QueryBuilder` 类允许你构建查询以检索符合特定条件的实体。通过组件查询，你可以在 `with_all` 和 `with_any` 方法中指定组件属性的条件。

### 语法

组件查询是一个 `Dictionary`，它将组件类映射到一个指定属性条件的查询 `Dictionary`。

```gdscript
{ ComponentClass: { property_name: { operator: value } } }
```

### 支持的运算符

*   `_eq`: 等于
*   `_ne`: 不等于
*   `_gt`: 大于
*   `_lt`: 小于
*   `_gte`: 大于或等于
*   `_lte`: 小于或等于
*   `_in`: 值在列表中
*   `_nin`: 值不在列表中

### 示例

#### 1\. 基本组件查询

检索 `C_TestC.value` 等于 `25` 的实体。

```gdscript
var result = QueryBuilder.new(world).with_all([
    { C_TestC: { "value": { "_eq": 25 } } }
]).execute()
```

#### 2\. 单个组件上的多个条件

检索 `C_TestC.value` 在 `20` 和 `25` 之间的实体。

```gdscript
var result = QueryBuilder.new(world).with_all([
    { C_TestC: { "value": { "_gte": 20, "_lte": 25 } } }
]).execute()
```

#### 3\. 结合组件查询和常规组件

检索具有 `C_TestD` 组件且 `C_TestC.value` 大于 `20` 的实体。

```gdscript
var result = QueryBuilder.new(world).with_all([
    C_TestD,
    { C_TestC: { "value": { "_gt": 20 } } }
]).execute()
```

#### 4\. 使用 `with_any` 与组件查询

检索 `C_TestC.value` 小于 `15` **或** `C_TestD.points` 大于或等于 `100` 的实体。

```gdscript
var result = QueryBuilder.new(world).with_any([
    { C_TestC: { "value": { "_lt": 15 } } },
    { C_TestD: { "points": { "_gte": 100 } } }
]).execute()
```

#### 5\. 使用 `_in` 和 `_nin` 运算符

检索 `C_TestC.value` 为 `10` 或 `25` 的实体。

```gdscript
var result = QueryBuilder.new(world).with_all([
    { C_TestC: { "value": { "_in": [10, 25] } } }
]).execute()
```

#### 6\. 复杂查询

检索满足以下条件的实体：

*   `C_TestC.value` 大于或等于 `25`，并且
*   `C_TestD.points` 大于 `75` **或** 小于 `30`，并且
*   排除具有 `C_TestE` 组件的实体。

```gdscript
var result = QueryBuilder.new(world).with_all([
    { C_TestC: { "value": { "_gte": 25 } } }
]).with_any([
    { C_TestD: { "points": { "_gt": 75 } } },
    { C_TestD: { "points": { "_lt": 30 } } }
]).with_none([C_TestE]).execute()
```

## 重要提示

*   **使用 ``with_none 的组件查询 ：组件查询**不支持**使用 `with_none` 方法。这是因为查询实体上不应存在的组件属性没有逻辑意义。使用 `with_none` 来排除具有某些组件的实体。``**
    
    ```gdscript
    # Correct usage of with_none
    var result = QueryBuilder.new(world).with_none([C_Inactive]).execute()
    ```
    
*   **空查询匹配组件的所有实例**
    
    如果你为一个组件提供一个空的查询字典，它将匹配所有具有该组件的实体，无论其属性如何。
    
    ```gdscript
    # This will match all entities that have C_TestC component
    var result = QueryBuilder.new(world).with_all([
        { C_TestC: {} }
    ]).execute()
    ```
    
*   **不存在的属性**
    
    如果你查询组件上不存在的属性，它将不会匹配任何实体。
    
    ```gdscript
    # Assuming 'non_existent' is not a property of C_TestC
    var result = QueryBuilder.new(world).with_all([
        { C_TestC: { "non_existent": { "_eq": 10 } } }
    ]).execute()
    # result will be empty
    ```
    

## 综合示例

这是一个展示多个组件查询的完整示例：

```gdscript
# Setting up entities with components
var entity1 = Entity.new()
entity1.add_component(C_TestC.new(25))
entity1.add_component(C_TestD.new(100))

var entity2 = Entity.new()
entity2.add_component(C_TestC.new(10))
entity2.add_component(C_TestD.new(50))

var entity3 = Entity.new()
entity3.add_component(C_TestC.new(25))
entity3.add_component(C_TestD.new(25))

var entity4 = Entity.new()
entity4.add_component(C_TestC.new(30))

world.add_entity(entity1)
world.add_entity(entity2)
world.add_entity(entity3)
world.add_entity(entity4)

# Query: Entities with C_TestC.value == 25 and C_TestD.points > 50
var result = QueryBuilder.new(world).with_all([
    { C_TestC: { "value": { "_eq": 25 } } },
    { C_TestD: { "points": { "_gt": 50 } } }
]).execute()
# result will include entity1
```

## 结论

组件查询通过允许您根据组件数据过滤实体，扩展了 GECS 框架的查询能力。通过使用支持的运算符，并将组件查询与传统组件过滤器结合使用，您可以精确地定位游戏逻辑所需的实体。

有关如何使用 `QueryBuilder` 的更多信息，请参阅 `query_builder.gd` 文档和 `test_query_builder.gd` 中的测试用例。