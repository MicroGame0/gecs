# Component Queries in GECS
GECS 中的组件查询

> **Advanced property-based entity filtering
> 基于属性的高级实体过滤**

Component Queries provide a powerful way to filter entities not just based on the presence of components but also on the data within those components. This allows for precise, data-driven entity selection in your game systems.
组件查询提供了一种强大的方式，不仅可以根据组件的存在，还可以根据组件中的数据来过滤实体。这使得在游戏系统中进行精确的数据驱动实体选择成为可能。

## 📋 Prerequisites
📋 先决条件

*   Understanding of [Core Concepts](CORE_CONCEPTS-zh-CN-dual.md)
    理解核心概念
*   Familiarity with [Basic Queries](CORE_CONCEPTS-zh-CN-dual.md#query-system)
    熟悉基本查询

## 🎯 Introduction
🎯 介绍

In standard ECS queries, you filter entities by which components they have or don't have. Component Queries take this further by letting you filter based on the **values** inside those components.
在标准的 ECS 查询中，您通过实体所具有的或不具有的组件来进行过滤。组件查询更进一步，允许您根据这些组件内的值来进行过滤。

Instead of just asking "which entities have a HealthComponent?", you can ask "which entities have a HealthComponent with current health less than 20?"
而不仅仅是问“哪些实体具有 HealthComponent？”，您还可以问“哪些实体具有 HealthComponent 且当前生命值小于 20？”

## Using Component Queries with `QueryBuilder`
使用组件查询 `QueryBuilder`

The `QueryBuilder` class allows you to construct queries to retrieve entities that match certain criteria. With component queries, you can specify conditions on component properties within `with_all` and `with_any` methods.
The `QueryBuilder` 类允许你构建查询以检索符合某些条件的实体。通过组件查询，你可以在 `with_all` 和 `with_any` 方法中指定组件属性的条件。

### Syntax
语法

A component query is a `Dictionary` that maps a component class to a query `Dictionary` specifying property conditions.
组件查询是一个 `Dictionary` ，它将组件类映射到一个查询 `Dictionary` ，该查询指定了属性条件。

```gdscript
{ ComponentClass: { property_name: { operator: value } } }
```

### Supported Operators
支持的操作符

*   `_eq`: Equal to
    `_eq` : 等于
*   `_ne`: Not equal to
    `_ne` : 不等于
*   `_gt`: Greater than
    `_gt` : 大于
*   `_lt`: Less than
    `_lt` : 小于
*   `_gte`: Greater than or equal to
    `_gte` : 大于或等于
*   `_lte`: Less than or equal to
    `_lte` : 小于或等于
*   `_in`: Value is in a list
    `_in` : 值在列表中
*   `_nin`: Value is not in a list
    `_nin` : 值不在列表中

### Examples
示例

#### 1\. Basic Component Query
1\. 基本组件查询

Retrieve entities where `C_TestC.value` is equal to `25`.
检索 `C_TestC.value` 等于 `25` 的实体。

```gdscript
var result = QueryBuilder.new(world).with_all([
    { C_TestC: { "value": { "_eq": 25 } } }
]).execute()
```

#### 2\. Multiple Conditions on a Single Component
2\. 单个组件上的多个条件

Retrieve entities where `C_TestC.value` is between `20` and `25`.
在 `C_TestC.value` 之间检索实体 .

```gdscript
var result = QueryBuilder.new(world).with_all([
    { C_TestC: { "value": { "_gte": 20, "_lte": 25 } } }
]).execute()
```

#### 3\. Combining Component Queries and Regular Components
3\. 组件查询与常规组件结合使用

Retrieve entities that have `C_TestD` component and `C_TestC.value` greater than `20`.
具有 `C_TestD` 组件且 `C_TestC.value` 大于 `20` 的实体 .

```gdscript
var result = QueryBuilder.new(world).with_all([
    C_TestD,
    { C_TestC: { "value": { "_gt": 20 } } }
]).execute()
```

#### 4\. Using `with_any` with Component Queries
4\. 在组件查询中使用 `with_any`

Retrieve entities where `C_TestC.value` is less than `15` **or** `C_TestD.points` is greater than or equal to `100`.
检索实体，其中 `C_TestC.value` 小于 `15` 或 `C_TestD.points` 大于或等于 `100` 。

```gdscript
var result = QueryBuilder.new(world).with_any([
    { C_TestC: { "value": { "_lt": 15 } } },
    { C_TestD: { "points": { "_gte": 100 } } }
]).execute()
```

#### 5\. Using `_in` and `_nin` Operators
5\. 使用 `_in` 和 `_nin` 运算符

Retrieve entities where `C_TestC.value` is either `10` or `25`.
检索实体，其中 `C_TestC.value` 为 `10` 或 `25` 之一。

```gdscript
var result = QueryBuilder.new(world).with_all([
    { C_TestC: { "value": { "_in": [10, 25] } } }
]).execute()
```

#### 6\. Complex Queries
6\. 复杂查询

Retrieve entities where:
检索实体，其中：

*   `C_TestC.value` is greater than or equal to `25`, and
    `C_TestC.value` 大于或等于 `25` ，并且
*   `C_TestD.points` is greater than `75` **or** less than `30`, and
    `C_TestD.points` 大于 `75` 或小于 `30` ，并且
*   Excludes entities with `C_TestE` component.
    排除具有 `C_TestE` 组件的实体

```gdscript
var result = QueryBuilder.new(world).with_all([
    { C_TestC: { "value": { "_gte": 25 } } }
]).with_any([
    { C_TestD: { "points": { "_gt": 75 } } },
    { C_TestD: { "points": { "_lt": 30 } } }
]).with_none([C_TestE]).execute()
```

## Important Notes
重要说明

*   **Component Queries with `with_none`**: Component queries are **not supported** with the `with_none` method. This is because querying properties of components that should not exist on the entity doesn't make logical sense. Use `with_none` to exclude entities that have certain components.
    使用 `with_none` 查询组件：使用 `with_none` 方法不支持查询组件。因为查询实体上不应存在的组件属性在逻辑上没有意义。请使用 `with_none` 排除具有某些组件的实体。
    
    ```gdscript
    # Correct usage of with_none
    var result = QueryBuilder.new(world).with_none([C_Inactive]).execute()
    ```
    
*   **Empty Queries Match All Instances of the Component
    空查询匹配所有组件实例**
    
    If you provide an empty query dictionary for a component, it will match all entities that have that component, regardless of its properties.
    如果你为某个组件提供一个空查询字典，它将匹配所有具有该组件的实体，无论其属性如何。
    
    ```gdscript
    # This will match all entities that have C_TestC component
    var result = QueryBuilder.new(world).with_all([
        { C_TestC: {} }
    ]).execute()
    ```
    
*   **Non-existent Properties
    不存在的属性**
    
    If you query a property that doesn't exist on the component, it will not match any entities.
    如果查询的属性在组件中不存在，将不会匹配任何实体。
    
    ```gdscript
    # Assuming 'non_existent' is not a property of C_TestC
    var result = QueryBuilder.new(world).with_all([
        { C_TestC: { "non_existent": { "_eq": 10 } } }
    ]).execute()
    # result will be empty
    ```
    

## Comprehensive Example
综合示例

Here's a full example demonstrating several component queries:
这里有一个完整的示例，演示了多种组件查询：

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

## Conclusion
结论

Component Queries extend the querying capabilities of the GECS framework by allowing you to filter entities based on component data. By utilizing the supported operators and combining component queries with traditional component filters, you can precisely target the entities you need for your game's logic.
组件查询通过允许根据组件数据过滤实体，扩展了 GECS 框架的查询功能。通过使用支持的操作符，并将组件查询与传统的组件过滤器结合使用，您可以精确地为目标游戏逻辑所需的实体。

For more information on how to use the `QueryBuilder`, refer to the `query_builder.gd` documentation and the test cases in `test_query_builder.gd`.
如需了解如何使用 `QueryBuilder` ，请参阅 `query_builder.gd` 文档和 `test_query_builder.gd` 中的测试案例。
