# GECS 性能测试指南

> **GECS 开发者的框架级性能测试**

本文档解释了如何运行和解读 GECS 性能测试。这主要面向需要确保 GECS 保持高性能的框架开发者和贡献者。

**对于游戏开发者：** 请参阅[性能优化指南](PERFORMANCE_OPTIMIZATION-zh-CN-translation.md)以优化您的游戏。

## 📋 前置条件

*   GECS 框架开发环境
*   gdUnit4 测试框架
*   Godot 4.x
*   测试系统依赖：`s_performance_test.gd` 和 `s_complex_performance_test.gd` 在 tests/systems/

## 🎯 概述

GECS 性能测试套件为所有关键 ECS 操作提供全面的基准测试：

*   **实体操作** ：创建、销毁、世界管理
*   **组件操作** ：添加、移除、查找、索引
*   **查询性能** : 所有查询类型、缓存、复杂场景
*   **系统处理** : 单一/多个系统、不同规模
*   **数组操作** : 优化的集合操作（交集、并集、差集）
*   **集成测试** : 真实的游戏场景和压力测试

## 🚀 运行性能测试

### 前提条件

将 `GODOT_BIN` 环境变量设置为您的 Godot 可执行文件：

```bash
# Windows
setx GODOT_BIN "C:\path\to\godot.exe"

# Linux/Mac
export GODOT_BIN="/path/to/godot"
```

### 运行单个测试套件

```bash
# Entity performance tests
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_entities.gd

# Component performance tests
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_components.gd

# Query performance tests
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_queries.gd

# System performance tests
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_systems.gd

# Array operations performance tests
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_arrays.gd

# Integration performance tests
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_integration.gd
```

### 运行完整性能套件

```bash
# Run all performance tests with comprehensive reporting
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_master.gd

# Quick smoke test to verify basic performance
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_master.gd::test_performance_smoke_test
```

## 📊 测试规模

性能测试使用三种不同的规模：

*   **SMALL\_SCALE**: 100 个实体（用于细粒度测试）
*   **中规模** : 1,000 个实体（适用于典型游戏场景）
*   **大规模** : 10,000 个实体（适用于压力测试）

## ⏱️ 性能阈值

测试包含自动性能阈值检查：

### 实体操作

*   创建 100 个实体：< 10ms
*   创建 1,000 个实体：< 50ms
*   向世界添加 1,000 个实体：< 100ms

### 组件操作

*   向 100 个实体添加组件：< 10ms
*   向 1,000 个实体添加组件：< 75ms
*   在 1,000 个实体中查找组件：< 30ms

### 查询性能

*   对100个实体的简单查询：小于5毫秒
*   对1,000个实体的简单查询：小于20毫秒
*   对10,000个实体的简单查询：小于100毫秒
*   复杂查询：< 50ms

### 系统处理

*   处理 100 个实体：< 5ms
*   处理 1,000 个实体：< 30ms
*   处理 10,000 个实体：< 150ms

### 游戏循环性能

*   逼真的游戏帧（1,000 个实体）：< 16ms（目标60 FPS）

## 📈 理解结果

### 性能指标

每个测试提供：

*   **平均时间** ：多次运行的平均执行时间
*   **最短/最长时间** ：最佳和最差的执行时间
*   **标准差** : 性能一致性
*   **每秒操作数** : 吞吐量测量
*   **每项操作时间** : 单项处理时间

### 结果文件

性能结果会以时间戳保存到 `res://reports/`：

*   `entity_performance_results.json`
*   `component_performance_results.json`
*   `query_performance_results.json`
*   `system_performance_results.json`
*   `array_performance_results.json`
*   `integration_performance_results.json`
*   `complete_performance_results_[timestamp].json`

### 结果解读

**良好的性能指标：**

*   ✅ 每秒高操作数（简单操作>10,000）
*   ✅ 标准差低（性能稳定）
*   ✅ 实体数量线性扩展
*   ✅ 查询缓存命中率 >80%

**性能警告信号：**

*   ⚠️ 测试耗时持续超过 50ms
*   ⚠️ 实体数量呈指数级时间扩展
*   ⚠️ 标准差高（性能不稳定）
*   ⚠️ 缓存命中率低于50%

## 🔄 回归测试

为了监控性能随时间变化：

1.  **建立基线** ：运行完整测试套件并保存结果
2.  **定期测试** ：在重大变更后运行测试
3.  **对比结果** ：使用主测试套件的回归检查
4.  **设置警报** ：监控性能下降超过 20%

## 🎯 优化区域

根据测试结果，集中优化工作在：

1.  **查询性能** : 对游戏体验最为关键
2.  **组件操作** : 高频操作
3.  **数组操作** : 核心性能构建模块
4.  **系统处理** : 帧率关键
5.  **内存使用** : 大规模场景

## ⚠️ 常见问题

### 缺失依赖项

如果测试因缺失类错误而失败，请确保这些文件存在：

*   `addons/gecs/tests/systems/s_performance_test.gd`
*   `addons/gecs/tests/systems/s_complex_performance_test.gd`

### gdUnit4 设置

除了设置 `GODOT_BIN` 之外，还需确保：

*   在项目设置中启用 gdUnit4 插件
*   所有测试组件类都已正确定义

## 🔧 自定义性能测试

要创建自定义性能测试：

1.  扩展 `PerformanceTestBase`
2.  使用 `benchmark()` 方法进行计时
3.  设置适当的性能阈值
4.  包含在主测试套件中

示例：

```gdscript
extends PerformanceTestBase

func test_my_custom_operation():
    var my_test = func():
        # Your operation here
        pass

    benchmark("My_Custom_Test", my_test)
    assert_performance_threshold("My_Custom_Test", 10.0, "Custom operation too slow")
```

## 📚 相关文档

*   **[性能优化](PERFORMANCE_OPTIMIZATION-zh-CN-translation.md)** \- 以用户为中心的优化指南
*   **[最佳实践](BEST_PRACTICES-zh-CN-translation.md)** \- 编写高性能的 ECS 代码
*   **[核心概念](CORE_CONCEPTS-zh-CN-translation.md)** \- 理解 ECS 架构

* * *

*该性能测试框架确保 GECS 在代码库演进过程中保持高性能。它是框架开发和优化工作的重要工具。*