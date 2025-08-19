# GECS Performance Testing Guide
GECS 性能测试指南

> **Framework-level performance testing for GECS developers
> GECS 开发者的框架级性能测试**

This document explains how to run and interpret the GECS performance tests. This is primarily for framework developers and contributors who need to ensure GECS maintains high performance.
本文档解释了如何运行和解读 GECS 性能测试。这主要面向需要确保 GECS 保持高性能的框架开发者和贡献者。

**For game developers:** See [Performance Optimization Guide](PERFORMANCE_OPTIMIZATION.md) for optimizing your games.
游戏开发者：请参阅性能优化指南以优化您的游戏。

## 📋 Prerequisites
📋 前置条件

*   GECS framework development environment
    GECS 框架开发环境
*   gdUnit4 testing framework
    gdUnit4 测试框架
*   Godot 4.x
*   Test system dependencies: `s_performance_test.gd` and `s_complex_performance_test.gd` in tests/systems/
    测试系统依赖： `s_performance_test.gd` 和 `s_complex_performance_test.gd` 在 tests/systems/

## 🎯 Overview
🎯 概述

The GECS performance test suite provides comprehensive benchmarking for all critical ECS operations:
GECS 性能测试套件为所有关键 ECS 操作提供全面的基准测试：

*   **Entity Operations**: Creation, destruction, world management
    实体操作：创建、销毁、世界管理
*   **Component Operations**: Addition, removal, lookup, indexing
    组件操作：添加、移除、查找、索引
*   **Query Performance**: All query types, caching, complex scenarios
    查询性能：所有查询类型、缓存、复杂场景
*   **System Processing**: Single/multiple systems, different scales
    系统处理：单/多系统、不同规模
*   **Array Operations**: Optimized set operations (intersect, union, difference)
    数组操作：优化集合操作（交集、并集、差集）
*   **Integration Tests**: Realistic game scenarios and stress tests
    集成测试：真实游戏场景和压力测试

## 🚀 Running Performance Tests
🚀 运行性能测试

### Prerequisites
前提条件

Set the `GODOT_BIN` environment variable to your Godot executable:
将 `GODOT_BIN` 环境变量设置为您的 Godot 可执行文件：

```bash
# Windows
setx GODOT_BIN "C:\path\to\godot.exe"

# Linux/Mac
export GODOT_BIN="/path/to/godot"
```

### Running Individual Test Suites
运行单个测试套件

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

### Running Complete Performance Suite
运行完整性能套件

```bash
# Run all performance tests with comprehensive reporting
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_master.gd

# Quick smoke test to verify basic performance
addons/gdUnit4/runtest.cmd -a res://addons/gecs/tests/performance/performance_test_master.gd::test_performance_smoke_test
```

## 📊 Test Scales
📊 测试规模

The performance tests use three different scales:
性能测试使用三种不同的规模：

*   **SMALL\_SCALE**: 100 entities (for fine-grained testing)
    SMALL\_SCALE: 100 个实体（用于细粒度测试）
*   **MEDIUM\_SCALE**: 1,000 entities (for typical game scenarios)
    中规模：1,000 个实体（适用于典型游戏场景）
*   **LARGE\_SCALE**: 10,000 entities (for stress testing)
    大规模：10,000 个实体（适用于压力测试）

## ⏱️ Performance Thresholds
⏱️ 性能阈值

The tests include automatic performance threshold checking:
测试包括自动性能阈值检查：

### Entity Operations
实体操作

*   Create 100 entities: < 10ms
    创建 100 个实体：< 10ms
*   Create 1,000 entities: < 50ms
    创建 1,000 个实体：< 50ms
*   Add 1,000 entities to world: < 100ms
    向世界添加 1,000 个实体：< 100ms

### Component Operations
组件操作

*   Add components to 100 entities: < 10ms
    向 100 个实体添加组件：< 10ms
*   Add components to 1,000 entities: < 75ms
    向 1,000 个实体添加组件：< 75ms
*   Component lookup in 1,000 entities: < 30ms
    在 1,000 个实体中查找组件：< 30ms

### Query Performance
查询性能

*   Simple query on 100 entities: < 5ms
    对 100 个实体的简单查询：小于 5 毫秒
*   Simple query on 1,000 entities: < 20ms
    对 1,000 个实体的简单查询：小于 20 毫秒
*   Simple query on 10,000 entities: < 100ms
    对 10,000 个实体的简单查询：小于 100 毫秒
*   Complex queries: < 50ms
    复杂查询：< 50ms

### System Processing
系统处理

*   Process 100 entities: < 5ms
    处理 100 个实体：< 5ms
*   Process 1,000 entities: < 30ms
    处理 1,000 个实体：< 30ms
*   Process 10,000 entities: < 150ms
    处理 10,000 个实体：< 150ms

### Game Loop Performance
游戏循环性能

*   Realistic game frame (1,000 entities): < 16ms (60 FPS target)
    逼真的游戏帧（1,000 个实体）：< 16ms（目标 60 FPS）

## 📈 Understanding Results
📈 理解结果

### Performance Metrics
性能指标

Each test provides:
每个测试提供：

*   **Average Time**: Mean execution time across multiple runs
    平均时间：多次运行的平均执行时间
*   **Min/Max Time**: Best and worst execution times
    最短/最长时间：最佳和最差的执行时间
*   **Standard Deviation**: Consistency of performance
    标准差：性能一致性
*   **Operations/Second**: Throughput measurement
    每秒操作数：吞吐量测量
*   **Time/Operation**: Per-item processing time
    每项操作时间：单项处理时间

### Result Files
结果文件

Performance results are saved to `res://reports/` with timestamps:
性能结果保存到 `res://reports/` ，并带有时间戳：

*   `entity_performance_results.json`
*   `component_performance_results.json`
*   `query_performance_results.json`
*   `system_performance_results.json`
*   `array_performance_results.json`
*   `integration_performance_results.json`
*   `complete_performance_results_[timestamp].json`

### Interpreting Results
解释结果

**Good Performance Indicators:
良好的性能指标：**

*   ✅ High operations/second (>10,000 for simple operations)
    ✅ 高操作数/秒（对于简单操作，>10,000）
*   ✅ Low standard deviation (consistent performance)
    ✅ 标准差低（性能稳定）
*   ✅ Linear scaling with entity count
    ✅ 实体数量线性扩展
*   ✅ Query cache hit rates >80%
    ✅ 查询缓存命中率 >80%

**Performance Warning Signs:
性能警告信号：**

*   ⚠️ Tests taking >50ms consistently
    ⚠️ 测试耗时持续超过 50ms
*   ⚠️ Exponential time scaling with entity count
    ⚠️ 实体数量呈指数级时间扩展
*   ⚠️ High standard deviation (inconsistent performance)
    ⚠️ 标准差高（性能不稳定）
*   ⚠️ Cache hit rates <50%
    ⚠️ 缓存命中率低于 50%

## 🔄 Regression Testing
🔄 回归测试

To monitor performance over time:
为了监控性能随时间变化：

1.  **Establish Baseline**: Run the complete test suite and save results
    建立基线：运行完整测试套件并保存结果
2.  **Regular Testing**: Run tests after significant changes
    定期测试：在重大变更后运行测试
3.  **Compare Results**: Use the master test suite's regression checking
    对比结果：使用主测试套件的回归检查
4.  **Set Alerts**: Monitor for >20% performance degradation
    设置警报：监控性能下降超过 20%

## 🎯 Optimization Areas
🎯 优化区域

Based on test results, focus optimization efforts on:
根据测试结果，将优化工作重点集中在：

1.  **Query Performance**: Most critical for gameplay
    查询性能：对游戏体验最为关键
2.  **Component Operations**: High frequency operations
    组件操作：高频操作
3.  **Array Operations**: Core performance building blocks
    数组操作：核心性能构建模块
4.  **System Processing**: Frame-rate critical
    系统处理：帧率关键
5.  **Memory Usage**: Large-scale scenarios
    内存使用：大规模场景

## ⚠️ Common Issues
⚠️ 常见问题

### Missing Dependencies
缺失依赖项

If tests fail with missing class errors, ensure these files exist:
如果测试因缺失类错误而失败，请确保这些文件存在：

*   `addons/gecs/tests/systems/s_performance_test.gd`
*   `addons/gecs/tests/systems/s_complex_performance_test.gd`

### gdUnit4 Setup
gdUnit4 设置

Beyond setting `GODOT_BIN`, ensure:
除了设置 `GODOT_BIN` 之外，还需确保：

*   gdUnit4 plugin is enabled in project settings
    gdUnit4 插件在项目设置中已启用
*   All test component classes are properly defined
    所有测试组件类都已正确定义

## 🔧 Custom Performance Tests
🔧 自定义性能测试

To create custom performance tests:
要创建自定义性能测试：

1.  Extend `PerformanceTestBase`
    扩展 `PerformanceTestBase`
2.  Use the `benchmark()` method for timing
    使用 `benchmark()` 方法进行计时
3.  Set appropriate performance thresholds
    设置适当的性能阈值
4.  Include in the master test suite
    包含在主测试套件中

Example:
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

## 📚 Related Documentation
📚 相关文档

*   **[Performance Optimization](PERFORMANCE_OPTIMIZATION.md)** - User-focused optimization guide
    性能优化 - 以用户为中心的优化指南
*   **[Best Practices](BEST_PRACTICES.md)** - Write performant ECS code
    最佳实践 - 编写高效的 ECS 代码
*   **[Core Concepts](CORE_CONCEPTS.md)** - Understanding the ECS architecture
    核心概念 - 理解 ECS 架构

* * *

*This performance testing framework ensures GECS maintains high performance as the codebase evolves. It's a critical tool for framework development and optimization efforts.
该性能测试框架确保 GECS 在代码库演进过程中保持高性能。它是框架开发和优化工作的重要工具。*